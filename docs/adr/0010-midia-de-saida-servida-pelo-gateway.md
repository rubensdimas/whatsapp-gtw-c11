# ADR 0010 — Mídia de saída servida pelo gateway

Data: 03/10/2026 · Status: Aceita

## Contexto

A Gupshup busca mídia por URL pública. O deploy citava storage, mas a arquitetura não tinha componente para isso.

## Decisão

Em E3, o gateway baixa o anexo do Chatwoot com as proteções de mídia, grava em volume durável próprio (MEDIA_STORAGE_PATH) e serve `GET /media/{token}` com token aleatório guardado como hash e validade MEDIA_URL_TTL_SECONDS (default 3600). Expirado o prazo, o arquivo é apagado por rotina de limpeza.

## Consequências

A URL do Chatwoot não é exposta ao provedor. O TTL deve ser ajustado após medir o tempo de busca da Gupshup. Object storage pode substituir o volume sem mudar o contrato.
