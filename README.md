# MBA DevOps + SRE com IA — Aplicação de Referência

Aplicação FastAPI usada como fio condutor em todos os módulos do curso.

## SLOs Definidos

| SLI | SLO | Error Budget |
|---|---|---|
| Availability | 99.9% | 43.2 min/mês |
| Latency p99 | < 500ms | — |
| Latency p95 | < 200ms | — |
| Error Rate | < 0.1% | — |

## Endpoints

- `GET /health` — Health check
- `GET /ready` — Readiness check (dependências)
- `GET /api/v1/items` — Lista itens (paginado)
- `GET /api/v1/items/{id}` — Detalhe de item
- `POST /api/v1/items` — Cria item
- `DELETE /api/v1/items/{id}` — Remove item

## Stack

- **Runtime:** Python 3.11+ / FastAPI / Uvicorn
- **Database:** PostgreSQL 16
- **Cache:** Redis 7
- **ORM:** SQLAlchemy 2.0 (async)
- **Migrations:** Alembic

## Quick Start

### Pré-requisitos

- Python 3.11+ (mesma versão usada no `Dockerfile`)
- Docker com Docker Compose
- git

### 1. Ambiente virtual e dependências

```bash
python3 -m venv .venv
source .venv/bin/activate
pip3 install --upgrade pip
pip3 install -r requirements.txt
```

> Com a venv ativa, `python3`, `pip3`, `alembic` e `uvicorn` apontam para `.venv/bin`.
> Ative a venv (`source .venv/bin/activate`) sempre que abrir um novo terminal.

### 2. Infraestrutura (Postgres + Redis)

```bash
docker compose up -d postgres redis
# ou, alternativamente:
./infra-local.sh up
```

> Use **apenas uma** das opções: ambas expõem o Postgres na porta 5432.
>
> `docker compose up -d` **sem argumentos** também sobe o container `app` na porta 8000
> (conflitando com o `uvicorn` local) e, se a migration falhar, ele cria as tabelas via
> `create_all`, fora do controle do Alembic.

### 3. Migrations

```bash
python3 -m alembic upgrade head
python3 -m alembic current   # deve mostrar: 001 (head)
```

> Use `python3 -m alembic` em vez de `alembic`: assim a raiz do projeto entra no
> `sys.path` e o `alembic/env.py` consegue importar o pacote `app`.

### 4. Rodar a aplicação

```bash
uvicorn app.main:app --reload --port 8000

# Em outro terminal
curl http://localhost:8000/health
curl http://localhost:8000/api/v1/items
```

Para rodar tudo em containers (app + Postgres + Redis + OTel Collector): `docker compose up -d`.

### 5. Testes

```bash
pytest
```

## Troubleshooting

| Sintoma | Causa | Solução |
|---|---|---|
| `alembic: command not found` | venv não ativada, ou dependências instaladas fora dela | `source .venv/bin/activate` e `pip3 install -r requirements.txt` |
| `ModuleNotFoundError: No module named 'app'` | `alembic` executado sem a raiz do projeto no `sys.path` | Rodar da raiz do projeto com `python3 -m alembic upgrade head` |
| `ModuleNotFoundError: No module named 'psycopg2'` | Dependências desatualizadas na venv | `pip3 install -r requirements.txt` |
| `DuplicateObject: type "itemstatus" already exists` | Tabelas criadas pelo fallback `create_all` da aplicação, sem registro no Alembic | `python3 -m alembic stamp head`, ou, em dev, `docker compose down -v` (**apaga os dados**) e refazer os passos 2 e 3 |
| `address already in use` (5432 ou 8000) | Postgres subido duas vezes (compose + `infra-local.sh`) ou container `app` ativo | `docker ps` e parar o container duplicado (`docker compose stop app`, `./infra-local.sh down`) |

## Estrutura

```
app/
├── app/
│   ├── __init__.py
│   ├── main.py          # FastAPI app, middleware, lifespan
│   ├── config.py        # Settings (env vars)
│   ├── database.py      # SQLAlchemy async session
│   ├── models.py         # SQLAlchemy models
│   ├── schemas.py       # Pydantic schemas
│   ├── crud.py           # CRUD operations
│   └── routers/
│       ├── __init__.py
│       ├── health.py    # /health, /ready
│       └── items.py     # /api/v1/items
├── alembic/
│   └── versions/
├── tests/
│   ├── __init__.py
│   └── test_items.py
├── docker-compose.yml
├── requirements.txt
├── alembic.ini
└── README.md
```

## Nota para o Curso

Esta aplicação é o **fio condutor** do curso. Cada módulo adiciona uma camada sobre ela:

| Módulo | O que é adicionado ao repositório |
|--------|-------------------------------------|
| **Módulo 1** | Dockerfile, Kubernetes manifests, docker-compose de produção, runbooks, design docs, skills |
| **Módulo 2** | CI/CD pipeline (GitHub Actions), security agent, prompt versioning |
| **Módulo 3** | OpenTelemetry instrumentation, SLI/SLO definitions, MCP servers config, Docker Compose de observabilidade (Prometheus + Grafana + LangFuse) |
| **Módulo 4** | Troubleshooting agent, RAG index sobre runbooks, ChatOps integration |
| **Módulo 5** | Webhook receptor (`/webhook/incident`), agente orquestrador (`operational_agent.py`), kill switch, audit log, modos copilot/semi-autônomo |

**O Módulo 5 NÃO começa do zero** — ele integra tudo que foi construído nos módulos anteriores. A aplicação FastAPI, os manifests, os runbooks, a instrumentação OTel e os MCP servers já existem. O que o Módulo 5 adiciona é a camada orquestradora que conecta tudo em um agente operacional end-to-end.

Tudo isso é gerado via prompt e construído progressivamente ao longo do curso.