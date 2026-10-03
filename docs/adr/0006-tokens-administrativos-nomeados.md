# ADR 0006 — Operação pela TI com tokens administrativos nomeados

Data: 03/10/2026 · Status: Aceita

## Contexto

Um ADMIN_API_TOKEN único não identifica o ator exigido pela auditoria de admin_command. A TI do Conselho opera o gateway: alertas, reconciliação e reprocessamento.

## Decisão

Cada pessoa da TI recebe token próprio, criado por `gateway admin-tokens create --name`, exibido uma única vez e guardado como hash. Tokens podem ser revogados e registram último uso. Todo comando administrativo grava o token/ator. Não há interface web de operação no MVP: REST, CLI, logs e métricas bastam.

## Consequências

SSO/OIDC fica para quando houver interface. A CLI roda dentro do container com acesso ao banco, sem expor endpoints extras.
