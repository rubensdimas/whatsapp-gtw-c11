# ADR 0011 — Tooling e ambiente de testes

Data: 03/10/2026 · Status: Aceita

## Decisão

Python 3.12; uv para dependências e lock; ruff para lint e formatação; mypy estrito em domínio e serviços; pytest com testcontainers (PostgreSQL real); httpx como cliente com respx para contratos HTTP; CI no GitHub Actions. docker compose de desenvolvimento com PostgreSQL e perfil opcional com `chatwoot/chatwoot:v4.10.1`, usado nos testes de contrato até existir a Inbox de homologação do Conselho.

## Consequências

Testes de contrato contra o Chatwoot local não substituem a homologação na instalação do Conselho.
