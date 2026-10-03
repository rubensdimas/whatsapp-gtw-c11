# ADR 0007 — Templates por comando em nota privada

Data: 03/10/2026 · Status: Aceita

## Contexto

A Inbox API não oferece seletor de templates. Exigir que a TI dispare cada template via REST não escala, e o endpoint usava ID do gateway, desconhecido pelo atendente.

## Decisão

- Nota privada de agente na inbox configurada que começa com `/template` ou `/templates` é um comando, nunca transportado como texto. É a única exceção à regra de ignorar notas privadas.
- `/templates` responde com nota privada listando chaves habilitadas e número de parâmetros.
- `/template <chave> <p1> | <p2> ...` valida chave e parâmetros no catálogo, aplica as regras de contato ativo (ADR 0004), cria mensagem outgoing espelhada com o texto renderizado e marcador gateway_template e agenda o envio. Erros voltam como nota privada.
- Qualquer agente da inbox pode usar no piloto; allowlist opcional de agentes no channel.
- Idempotência pelo ID da nota; ator é o autor da nota.
- Catálogo em tabela, gerido por `gateway templates add|disable|list`, com texto guardado para renderizar o espelho.
- O endpoint REST usa o display_id do Chatwoot: `POST /admin/channels/{slug}/conversations/{display_id}/templates`.

## Consequências

O filtro anti-loop do espelho depende de content_attributes (ou source_id) aparecer no webhook message_created da 4.10.1; verificar na Etapa 0. Notas privadas comuns continuam ignoradas.
