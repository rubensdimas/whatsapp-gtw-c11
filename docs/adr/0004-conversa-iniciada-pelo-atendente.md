# ADR 0004 — Conversa iniciada pelo atendente

Data: 03/10/2026 · Status: Aceita

## Contexto

O atendente pode criar uma conversa na Inbox API para um contato que nunca escreveu ao número. Não há channel_contact, wa_id mapeado nem janela aberta.

## Decisão

Ao receber outgoing de conversa desconhecida da inbox configurada, o gateway lê conversa e contato pela Application API. Se o contato tiver phone_number E.164 válido, deriva wa_id (dígitos) e cria channel_contact com o source_id do vínculo existente. Sem janela, texto livre recebe failed e aviso privado orientando o uso de `/template`. Sem telefone válido, failed e aviso específico. Conflito com channel_contact de outro contato para o mesmo wa_id gera failed e aviso para revisão.

## Consequências

O contato ativo passa a ser possível por template (ADR 0007). O primeiro inbound da pessoa reutiliza o mesmo channel_contact.
