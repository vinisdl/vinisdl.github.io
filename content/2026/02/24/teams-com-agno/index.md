---
title: "Como criar Teams no Agno: Orquestração multi-agente com arquitetura multi-tenant (passo a passo)"
date: 2026-02-24T17:28:45-0300
draft: false
description: "Aprenda a criar Teams no Agno com uma arquitetura de orchestrator e multi-tenant: definição de membros e roles, roteamento de tarefas, observabilidade e deploy local com exemplos de código."
tags:
  - agno
  - multi-tenant
  - agents
categories: []
---

No Agno, "Team" é um grupo de agentes com papéis definidos e um fluxo que decide quem faz o quê. Quando você coloca um orchestrator na frente e pensa em multi-tenant (vários clientes no mesmo sistema, com isolamento de dados), o desenho fica mais claro. Este post é um rascunho aplicável: convenções e passo a passo com código para você adaptar.

Referências usadas como base para o padrão de arquitetura:

- Repo: [agent-orchestrator-mult-tenant](https://github.com/vinisdl/agent-orchestrator-mult-tenant)
- Post: [Agno Agent UI self-hosted + RAG](https://vinisdl.github.io/2026/02/23/agno-agent-ui-self-hosted-rag/)

---

## Pré-requisitos

- Python 3.10+ (de preferência 3.11+)
- Agno instalado no projeto
- Docker + Docker Compose (opcional, para Postgres e serviços locais)
- Chave de LLM (OpenAI, Anthropic ou setup self-hosted)
- Postgres (recomendado para sessões, logs e histórico por tenant)
- Noções de multi-tenant (`tenant_id`, isolamento por schema ou coluna) e de observabilidade (logs estruturados e traces)

---

## Arquitetura: orchestrator + multi-tenant

### Visão geral

1. **API/Backend** — Recebe a requisição do usuário (com `tenant_id`).
2. **Orchestrator** — Valida o tenant, carrega a config do tenant (modelos, limites, ferramentas, base de conhecimento), escolhe ou monta um Team e roteia a tarefa para o membro certo.
3. **Team (Agno)** — Colaboração entre agentes (sequencial ou com delegação de subtarefas) e uso de ferramentas (RAG, web, DB, fila, integrações).
4. **Storage e observabilidade** — Persistência por tenant (sessões, memória, histórico); logs e traces com `tenant_id`, `team_id`, `run_id`.

### Multi-tenant na prática

Propague `tenant_id` em tudo: storage/sessões, índices e bases de conhecimento, logs/traces e chaves de cache. Mantenha a config por tenant numa tabela ou arquivo (modelo, temperatura, tools habilitadas, rate limits, base de conhecimento). Para isolamento: no mínimo uma coluna `tenant_id` nas tabelas; se fizer sentido, schema por tenant.

---

## Passo a passo: criando um Team no Agno

### 1. Contrato de entrada e saída do orchestrator

Defina uma entrada mínima e uma saída padronizada para não criar acoplamento invisível quando plugar UI, filas ou webhooks.

**Entrada:**

```python
from dataclasses import dataclass

@dataclass
class OrchestratorRequest:
    tenant_id: str
    user_id: str
    message: str
    conversation_id: str | None = None
    metadata: dict | None = None
```

**Saída:**

```python
@dataclass
class OrchestratorResponse:
    run_id: str
    team_id: str
    final_answer: str
    traces_url: str | None = None
```

### 2. Roles do Team

Cada membro com uma responsabilidade. Exemplo de roles comuns:

- **Router / Orchestrator** — Entende a intenção e distribui a tarefa.
- **RAG Researcher** — Busca na base de conhecimento do tenant.
- **Writer** — Monta a resposta final no tom da marca.
- **Tool Runner** — Chama integrações (CRM, tickets, DB, etc.).

### 3. Agentes (membros) com instruções claras

Instrução curta demais em Team vira cada agente "por si". Vale ser explícito.

Exemplo conceitual:

```python
def make_research_agent(tenant_cfg):
    return Agent(
        name="researcher",
        instructions=[
            "Você busca informações na base de conhecimento do tenant.",
            "Cite a fonte ou o trecho quando possível.",
            "Se não achar, diga que não encontrou e sugira próximos passos."
        ],
        model=tenant_cfg.model,
        tools=[tenant_cfg.knowledge_tool],
    )

def make_writer_agent(tenant_cfg):
    return Agent(
        name="writer",
        instructions=[
            "Escreva a resposta final em PT-BR, objetiva, com exemplos.",
            "Não invente detalhes. Se faltar dado, peça o mínimo para seguir.",
            "Use um tom prático, sem linguagem inflada."
        ],
        model=tenant_cfg.model,
    )
```

### 4. Montar o Team

```python
def build_team_for_tenant(tenant_cfg):
    researcher = make_research_agent(tenant_cfg)
    writer = make_writer_agent(tenant_cfg)

    return Team(
        name=f"support-team-{tenant_cfg.tenant_id}",
        members=[researcher, writer],
        instructions=[
            "Trabalhem em conjunto: primeiro busquem e confirmem fatos, depois respondam.",
            "Mantenham a resposta final curta e acionável."
        ],
    )
```

### 5. Roteamento de tarefas

Duas opções: roteamento fixo no código (simples e barato no começo) ou um agente "router" que classifica a intenção (mais flexível, mas exige validação de saída).

**Opção A — Roteamento hard-coded**

Bom para MVP e para controlar custo.

```python
def route(message: str) -> str:
    msg = message.lower()
    if "documentação" in msg or "como" in msg or "guia" in msg:
        return "research_then_write"
    if "erro" in msg or "bug" in msg:
        return "write_only"
    return "research_then_write"
```

Rotas típicas:

```python
ROUTES = {
    "research_then_write": ["researcher", "writer"],
    "write_only": ["writer"],
}
```

Execução (conceitual):

```python
def run_team(team, route_name, user_message):
    context = {"user_message": user_message}

    for member_name in ROUTES[route_name]:
        agent = team.get_member(member_name)
        result = agent.run(context)
        context[member_name] = result

    return context["writer"].final
```

**Opção B — Router agent**

Um agente classifica a intenção e devolve um plano (ex.: JSON). Valide a saída (ex.: JSON schema) e tenha um fallback.

### 6. Escalar membros e roles (multi-tenant)

Mantenha um catálogo de teams; cada tenant escolhe um template.

```python
TEAM_TEMPLATES = {
    "default_support": ["researcher", "writer"],
    "sales_assistant": ["product_specialist", "writer"],
    "triage": ["router", "researcher", "writer"],
}
```

No config do tenant:

```json
{
  "tenant_id": "acme",
  "team_template": "triage",
  "model": "gpt-4.1-mini",
  "knowledge_base": "kb_acme"
}
```

---

## Observabilidade (logs, traces e métricas)

Desde o dia 1, inclua: `tenant_id`, `team_id`, `run_id`, `conversation_id`, tempo total, tokens/custo por membro (se tiver) e a rota escolhida.

Exemplo de log estruturado:

```python
logger.info("team_run_started", extra={
    "tenant_id": req.tenant_id,
    "team_id": team.name,
    "run_id": run_id,
    "route": route_name,
})
```

Se usar OpenTelemetry, crie spans por etapa (orchestrator, team.run, cada agente) e injete `tenant_id` como atributo do span.

---

## Deploy local (Docker Compose)

Setup mínimo para dev: API (FastAPI/Flask), Postgres e, opcionalmente, Grafana/Tempo/Loki (ou outro stack de observabilidade).

Exemplo de `docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_USER: postgres
      POSTGRES_DB: agno
    ports:
      - "5432:5432"

  api:
    build: .
    environment:
      DATABASE_URL: postgresql+psycopg://postgres:postgres@postgres:5432/agno
    ports:
      - "8000:8000"
    depends_on:
      - postgres
```

---

## Dicas práticas

- Não deixe o Team decidir tudo sozinho. Comece com roteamento simples e aumente a autonomia aos poucos.
- Trate a config do tenant como parte do produto: versione, logue mudanças e tenha defaults.
- Defina budgets por tenant (limite de tokens/custo e timeout).
- Separe bem: orchestrator cuida de autenticação, tenant, roteamento e auditoria; o Team/Agno cuida da execução e das ferramentas.
- Falha controlada: se a base não responder, o agente deve dizer "não encontrei" em vez de inventar.
- Cache por tenant: embeddings, respostas frequentes, tools que chamam APIs externas.
