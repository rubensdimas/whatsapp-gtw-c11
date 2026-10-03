# ADR 0003 — Fila por contato com prazo de retenção

Data: 03/10/2026 · Status: Aceita

## Contexto

A ordenação por contato impedia iniciar a mensagem seguinte enquanto a anterior estivesse em resultado ambíguo. Sem limite, um único timeout bloquearia o atendimento daquele contato até intervenção da TI.

## Decisão

Uma parte em reconciliation_required retém a fila de saída do contato por ORDERING_HOLD_SECONDS (default 300). Se um recibo resolver a ambiguidade nesse prazo, a fila segue normalmente. Ao expirar, a mensagem recebe PATCH failed com external_error "Envio incerto — verifique antes de reenviar", um aviso privado é criado e a fila é liberada. A tentativa local permanece reconciliation_required para análise da TI; nada é reenviado automaticamente.

## Consequências

A ordem é garantida no caso comum de recibo atrasado e o impacto de falha real fica limitado. Se um recibo posterior comprovar entrega, o gateway registra e tenta atualizar o status no Chatwoot; a aceitação da transição failed → delivered pela 4.10.1 deve ser verificada em homologação.
