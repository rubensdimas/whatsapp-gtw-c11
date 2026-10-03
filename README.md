# CREFITO-11 — WhatsApp Gateway

Documentação inicial do MVP. Provedor inicial: **Gupshup**. Interface de atendimento humano: **Chatwoot 4.10.1**, por Inbox API. Data de referência: 03/10/2026.

Este pacote contém especificações para iniciar o desenvolvimento; não contém uma aplicação executável. Os exemplos utilizam valores fictícios. Compatibilidade no código upstream não substitui homologação na instalação do Conselho.

## Objetivo

Receber mensagens do WhatsApp no Chatwoot, enviar respostas dos atendentes pela Gupshup e manter correlação e status de entrega. O gateway será independente do fornecedor por meio de um adapter, implementando inicialmente somente Gupshup.

## Ordem de leitura

| Arquivo | Finalidade |
| --- | --- |
| [AGENTS.md](AGENTS.md) | Orientações para desenvolvimento com assistentes de código |
| [docs/01-ESCOPO.md](docs/01-ESCOPO.md) | Requisitos, limites e critérios de aceite |
| [docs/02-ARQUITETURA.md](docs/02-ARQUITETURA.md) | Componentes, fluxos e decisões técnicas |
| [docs/03-CONTRATOS-API.md](docs/03-CONTRATOS-API.md) | Integrações Gupshup e Chatwoot 4.10.1 |
| [docs/04-MODELO-DE-DADOS.md](docs/04-MODELO-DE-DADOS.md) | Entidades, índices e correlação |
| [docs/05-CONFIABILIDADE-E-SEGURANCA.md](docs/05-CONFIABILIDADE-E-SEGURANCA.md) | Deduplicação, retentativas e proteção |
| [docs/06-CONFIGURACAO-E-OPERACAO.md](docs/06-CONFIGURACAO-E-OPERACAO.md) | Preparação, configuração e operação |
| [docs/07-ROADMAP.md](docs/07-ROADMAP.md) | Sequência de implementação e entregas |
| [docs/08-HOMOLOGACAO.md](docs/08-HOMOLOGACAO.md) | Cenários e evidências de validação |
| [docs/09-REFERENCIAS.md](docs/09-REFERENCIAS.md) | Fontes oficiais e compatibilidade verificada |
| [docs/adr/](docs/adr/README.md) | Decisões registradas e seus motivos |

## Premissas adotadas

FastAPI, PostgreSQL, SQLAlchemy/Alembic e worker dedicado. API e worker compartilham uma base de código e têm processos separados. PostgreSQL oferece a fila persistente inicial; Redis não é necessário para o primeiro MVP. Python 3.12, uv, ruff, mypy, pytest e testcontainers ([ADR 0011](docs/adr/0011-tooling.md)). Containers, Compose local e implantação em Swarm/Traefik, no mesmo cluster do Chatwoot.

Canais ficam no banco ([ADR 0001](docs/adr/0001-canais-no-banco.md)); produção opera um número. A TI do Conselho opera o gateway com tokens nomeados ([ADR 0006](docs/adr/0006-tokens-administrativos-nomeados.md)); atendentes enviam templates por comando em nota privada ([ADR 0007](docs/adr/0007-comando-de-template-em-nota-privada.md)).

IA, Agno, RAG, autenticação de profissionais, Implanta.NET, negociação automática e workflows n8n ficam para outra entrega. A autenticação técnica do gateway e das APIs integra o MVP atual.

## Primeiro passo

Iniciar a Etapa 1 (fundação) do roadmap, que não depende de contratos externos, em paralelo à Etapa 0: vincular o número de teste ao app Gupshup existente, capturar payloads do sandbox e testar contratos Chatwoot no compose com 4.10.1 ([ADR 0012](docs/adr/0012-etapa-1-em-paralelo.md)). Nenhuma mudança de provedor no número de produção está autorizada por estes documentos.
