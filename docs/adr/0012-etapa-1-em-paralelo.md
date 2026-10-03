# ADR 0012 — Etapa 1 em paralelo à Etapa 0

Data: 03/10/2026 · Status: Aceita

## Contexto

O app Gupshup existe, mas o número de teste ainda não foi vinculado. Não há Inbox de homologação nem ambiente Swarm de homologação confirmados.

## Decisão

A Etapa 1 começa imediatamente, pois não depende de contratos externos. A Etapa 2 pode começar com fixtures sintéticas marcadas como provisórias. Após a Etapa 1, capturar payloads reais do sandbox Gupshup por túnel HTTPS temporário apontando para o gateway local; estes valem como "reais de sandbox" e serão reconferidos com o número de teste. Testes Chatwoot usam o compose com 4.10.1.

## Consequências

Fixtures provisórias são substituídas antes do piloto E1. Divergência entre sandbox e número vinculado é tratada como mudança de contrato.
