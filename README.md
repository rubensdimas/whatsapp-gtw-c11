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

## Premissas adotadas

FastAPI, PostgreSQL, SQLAlchemy/Alembic e worker dedicado. API e worker compartilham uma base de código e têm processos separados. PostgreSQL oferece a fila persistente inicial; Redis não é necessário para o primeiro MVP. Containers, Compose local e implantação compatível com Swarm/Traefik são a direção proposta, a validar no ambiente real.

IA, Agno, RAG, autenticação de profissionais, Implanta.NET, negociação automática e workflows n8n ficam para outra entrega. A autenticação técnica do gateway e das APIs integra o MVP atual.

## Primeiro passo

Executar a etapa de contratos descrita no roadmap: criar uma Inbox API de homologação no Chatwoot 4.10.1, preparar um app/número Gupshup de teste e capturar payloads reais anonimizados. Nenhuma mudança de provedor no número de produção está autorizada por estes documentos.
