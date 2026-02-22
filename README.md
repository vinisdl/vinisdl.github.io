# Blog - Marco Dall'Alba

Template de blog estático com [Hugo](https://gohugo.io/) e tema [Hextra](https://github.com/imfing/hextra).

## Desenvolvimento Local

### Pré-requisitos

**Docker (recomendado)**

- Docker e Docker Compose

**Instalação local**

- Hugo (versão Extended)
- Go
- Ruby
- Git

### Usando Docker

1. **Inicie o ambiente:**

```shell
./scripts/dev.sh start
```

2. **Acesse o blog:**

- <http://localhost:1313>

3. **Comandos úteis:**

```shell
./scripts/dev.sh logs           # Ver logs
./scripts/dev.sh stop           # Parar ambiente
./scripts/dev.sh new-post "Título do Post"  # Criar novo post
./scripts/dev.sh generate-index # Gerar índice
./scripts/dev.sh help           # Ver todos os comandos
```

### Instalação local

```shell
# Gerar índice
./scripts/generate_index.rb

# Rodar servidor
hugo server --logLevel debug --disableFastRender -p 1313
```

## Criando posts

```shell
# Com Docker
./scripts/dev.sh new-post "Título do Post"

# Manualmente
mkdir -p content/2025/02/21/meu-post
# Crie content/2025/02/21/meu-post/index.md com frontmatter
```

### Estrutura de um post

```markdown
---
title: "Título do Post"
date: 2025-02-21T10:00:00-03:00
draft: false
description: "Descrição do post"
tags: [tag1, tag2]
---

Conteúdo do post aqui...
```

## Estrutura do projeto

```
├── content/           # Posts e páginas (Markdown)
│   ├── _index.md       # Índice da home (gerado por script)
│   ├── about.md       # Página Sobre
│   └── YYYY/MM/DD/slug/
├── layouts/            # Templates e partials
├── hugo.yaml           # Configuração do Hugo
├── scripts/            # Scripts (generate_index.rb, dev.sh)
├── Dockerfile
└── docker-compose.yml
```

## Personalização

- **Nome e autor:** `hugo.yaml` → `title`, `params.author`, `params.description`
- **URL em produção:** `hugo.yaml` → `baseURL`
- **Redes sociais:** `hugo.yaml` → `menu.main` e `menu.sidebar`
- **Comentários Disqus:** `hugo.yaml` → `services.disqus.shortname`
- **Página Sobre:** edite `content/about.md`

## Licença

Conforme definido no projeto original (ex.: CC BY-NC-SA 4.0). Ajuste conforme sua preferência.
