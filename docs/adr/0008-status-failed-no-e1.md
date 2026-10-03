# ADR 0008 — PATCH failed e aviso privado no E1

Data: 03/10/2026 · Status: Aceita

## Contexto

H09 (E1) exige failed e aviso privado, mas o roadmap só implementava PATCH de status na Etapa 4.

## Decisão

A Etapa 3 implementa PATCH failed com external_error sanitizado e aviso privado deduplicado. A Etapa 4 adiciona sent/delivered/read, recibos pendentes, progressão e agregação.

## Consequências

O piloto E1 já informa bloqueios e falhas ao atendente; recibos positivos continuam sem promessa até E2.
