# ADR 0001 — Canais configurados no banco

Data: 03/10/2026 · Status: Aceita

## Contexto

As variáveis de ambiente previam um único canal (CHATWOOT_INBOX_ID, GUPSHUP_APP_NAME, GUPSHUP_SOURCE), enquanto o modelo de dados já tinha a entidade channel e channel_id em todas as chaves. Havia duas fontes de verdade para a mesma configuração.

## Decisão

A tabela channel é a única fonte de verdade de app Gupshup, número, conta e inbox. Canais são criados e alterados por CLI (`gateway channels ...`). Produção opera um canal; o código não assume canal único. O ambiente guarda apenas segredos de provedor e configurações globais; a chave Gupshup de cada canal é lida de `GUPSHUP_API_KEY__<SLUG>`.

## Consequências

Rotas de webhook carregam o slug do canal (ADR 0005). Adicionar um número futuro é cadastro, não mudança de código. Testes devem cobrir dois canais para evitar vazamento de escopo.
