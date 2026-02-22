---
date: 2026-02-22T02:13:34-0300
draft: false
title: "Arquitetura SaaS multi-tenant: um projeto de referência em .NET e React"
slug: arquitetura-saas-multitenant-referencia
tags:
  - saas
  - arquitetura
  - dotnet
  - react
  - multitenancy
  - keycloak
  - observabilidade
description: "Conheça um projeto de referência para aplicações SaaS multi-tenant: backend .NET 10, frontend React 19, PostgreSQL, Keycloak, RabbitMQ e observabilidade com OpenTelemetry e Grafana."
---

Montar um SaaS do zero envolve uma porção de decisões que se repetem: como isolar dados por cliente, como fazer auth, como observar o sistema em produção. Fica fácil perder tempo reinventando a roda ou grudando peças que não conversam. Por isso criei um repositório de **arquitetura de referência** que junta, em um único lugar, as peças que costumam aparecer em um SaaS multi-tenant moderno — e que você pode clonar, adaptar e usar como ponto de partida.

Neste post mostro o que esse projeto oferece, as escolhas técnicas e como subir tudo localmente.

## O que é o SaaS Architecture

O [**SaaS Architecture**](https://github.com/vinisdl/saas-architecture) é um repositório aberto que implementa uma stack completa para aplicações SaaS multi-tenant. O objetivo é servir de **template** e **referência**: em vez de decidir sozinho cada camada, você tem um desenho coerente — backend, frontend, banco, autenticação, filas e observabilidade — pronto para evoluir.

Principais características:

- **Multi-tenancy** com isolamento por tenant (TenantId, contexto por request).
- **Autenticação e autorização** via Keycloak (OAuth2/OIDC, JWT).
- **Backend** em arquitetura hexagonal + CQRS (MediatR).
- **Frontend** SPA em React com roteamento e área administrativa.
- **Mensageria** com RabbitMQ para eventos entre serviços.
- **Observabilidade** com OpenTelemetry (OTLP) e integração Grafana Cloud / Grafana Alloy.

Ou seja: não é um "hello world" — é uma base real para você estender com regras de negócio e domínio.

## Stack e decisões técnicas

A tabela abaixo resume a stack; em seguida comento o porquê das escolhas.

| Camada          | Tecnologia                |
| --------------- | ------------------------- |
| Banco de dados  | PostgreSQL 16             |
| Backend         | C# 14 (.NET 10), ASP.NET Core |
| Frontend        | React 19, Vite 7          |
| Autenticação    | Keycloak (OIDC/JWT)       |
| Mensageria      | RabbitMQ                  |
| Observabilidade | OpenTelemetry, Grafana Alloy |
| Infraestrutura  | Docker, Docker Compose    |

**Backend em .NET 10** — Ecossistema maduro, performance e tipagem forte (C# 14). A combinação hexagonal + CQRS com MediatR mantém o domínio isolado e os use cases explícitos, o que escala bem quando o produto cresce.

**React 19 + Vite 7** — Frontend enxuto e rápido. Vite entrega HMR e build otimizado; React 19 traz melhorias recentes de runtime e DX. O front consome a API e pode enviar telemetria (traces) via proxy para o backend, que encaminha ao Alloy.

**Keycloak** — Auth e identidade sem depender de serviço proprietário. OIDC/JWT é padrão de mercado; você pode trocar por outro IdP depois se quiser, mantendo o mesmo contrato (tokens JWT).

**RabbitMQ** — Filas para eventos assíncronos entre serviços (ex.: notificações, integrações). Desacopla processos e permite escalar consumidores independentemente.

**OpenTelemetry + Grafana Alloy** — Telemetria padronizada (traces, métricas, logs). O Alloy recebe OTLP do backend (e opcionalmente do front via proxy) e pode enviar para Grafana Cloud ou qualquer backend compatível. Você ganha visibilidade desde o primeiro dia.

## Arquitetura na prática

![Diagrama da arquitetura: navegador, frontend React, backend .NET, PostgreSQL, Keycloak, RabbitMQ, Grafana Alloy e Grafana Cloud](https://mermaid.ink/img/Zmxvd2NoYXJ0IFRCCiAgc3ViZ3JhcGggdXN1YXJpbyBbVXN1w6FyaW9dCiAgICBCcm93c2VyW05hdmVnYWRvcl0KICBlbmQKICBzdWJncmFwaCBhcGxpY2FjYW8gW0FwbGljYcOnw6NvXQogICAgRnJvbnRlbmRbRnJvbnRlbmQgUmVhY3RdCiAgICBCYWNrZW5kW0JhY2tlbmQgLk5FVCBBUEldCiAgZW5kCiAgc3ViZ3JhcGggZGFkb3MgW0RhZG9zIGUgQXV0aF0KICAgIFBvc3RncmVzWyhQb3N0Z3JlU1FMKV0KICAgIEtleWNsb2FrW0tleWNsb2FrXQogIGVuZAogIHN1YmdyYXBoIG1lbnNhZ2VyaWEgW01lbnNhZ2VyaWFdCiAgICBSYWJiaXRNUVtSYWJiaXRNUV0KICBlbmQKICBzdWJncmFwaCBvYnNlcnZhYmlsaWRhZGUgW09ic2VydmFiaWxpZGFkZV0KICAgIEFsbG95W0dyYWZhbmEgQWxsb3ldCiAgZW5kCiAgc3ViZ3JhcGggY2xvdWQgW0dyYWZhbmEgQ2xvdWRdCiAgICBUZW1wb1tUcmFjZXNdCiAgICBNaW1pcltNZXRyaWNzXQogIGVuZAogIEJyb3dzZXIgLS0-IEZyb250ZW5kCiAgRnJvbnRlbmQgLS0-fEhUVFAgLyBBUEl8IEJhY2tlbmQKICBGcm9udGVuZCAtLT58T1RMUCBwcm94eXwgQmFja2VuZAogIEJhY2tlbmQgLS0-IFBvc3RncmVzCiAgQmFja2VuZCAtLT4gS2V5Y2xvYWsKICBCYWNrZW5kIC0tPiBSYWJiaXRNUQogIEJhY2tlbmQgLS0-fE9UTFAgZ1JQQ3wgQWxsb3kKICBCYWNrZW5kIC0tPnxPVExQIEhUVFAgcHJveHl8IEFsbG95CiAgQWxsb3kgLS0-IFRlbXBvCiAgQWxsb3kgLS0-IE1pbWly)

O fluxo geral é: usuário acessa o frontend; o front chama a API e, se configurado, envia traces para o backend (proxy), que repassa ao Alloy. O backend fala com PostgreSQL, Keycloak (validação de token) e RabbitMQ, e envia telemetria direto ao Alloy; o Alloy encaminha para Grafana Cloud (Traces, Métricas) conforme a configuração.

Pontos que valem destacar:

- **Multi-tenancy** — O TenantId entra no contexto da request (middleware/claims). As queries e regras de negócio usam esse contexto para isolar dados por tenant. Não é "um DB por cliente", e sim um modelo de tenant compartilhado com isolamento lógico, que é o mais comum em SaaS.
- **Auth** — Keycloak emite os JWTs; o backend valida e extrai claims (incluindo tenant quando aplicável). O front usa um client OIDC para login e envia o token nas chamadas à API.
- **Observabilidade** — Tudo passa por OTLP (gRPC ou HTTP). Quem quiser só testar localmente pode subir o Alloy sem Grafana Cloud; quem quiser produção pode configurar as variáveis do Alloy para enviar para a nuvem.

A documentação do repositório traz diagramas (incluindo flowchart da arquitetura), configuração do Keycloak e passos para Grafana Cloud.

## Como rodar

Pré-requisitos: Docker e Docker Compose. Opcional: .NET 10 SDK e Node.js LTS se quiser rodar backend ou frontend fora do Docker.

**Produção (stack completa):**

```bash
git clone https://github.com/vinisdl/saas-architecture.git
cd saas-architecture
docker compose -f docker-compose.prd.yml up -d
```

**Desenvolvimento (hot reload no backend e frontend):**

```bash
docker compose -f docker-compose.dev.yml up --build
```

Alterações em C# disparam recompilação e reinício do backend; alterações no front disparam HMR sem full reload.

Após subir, você tem:

| Serviço       | URL                     |
| ------------- | ----------------------- |
| Frontend      | http://localhost:3000   |
| API           | http://localhost:5000   |
| Keycloak      | http://localhost:8080    |
| RabbitMQ      | http://localhost:15672   |
| Alloy (OTLP)  | portas 4317 (gRPC), 4318 (HTTP) |

Para enviar telemetria ao Grafana Cloud, basta configurar as variáveis do Alloy conforme a documentação em `docs/`.

O [SaaS Architecture](https://github.com/vinisdl/saas-architecture) junta, em um único repo, as peças que a maioria dos times acaba montando ao longo do tempo: multi-tenancy, auth com Keycloak, backend estruturado em .NET 10, frontend em React, filas com RabbitMQ e observabilidade com OpenTelemetry e Grafana. Serve tanto como **referência** para estudar decisões quanto como **base** para um novo produto.

Se você está pensando em montar um SaaS ou quer um exemplo "de verdade" para comparar com o que está fazendo, vale dar uma olhada — e se fizer sentido, dar uma estrela no repo ou abrir uma issue com sugestões. Para mais conteúdo sobre arquitetura e desenvolvimento, acompanhe o blog e o [LinkedIn](https://www.linkedin.com/in/vinisdl/).
