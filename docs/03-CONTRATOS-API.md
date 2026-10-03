# Contratos das APIs

Links oficiais estão em [referências](09-REFERENCIAS.md). Exemplos são sintéticos. A homologação deve capturar os formatos reais da conta e da tag 4.10.1; não copiar payloads de outra geração de API.

## Gupshup: produto selecionado

Usar WhatsApp Self-Serve Messaging API, envio /wa/api/v1 e callback **V2**. A versão do envelope não é a versão do endpoint de envio. Não misturar com Partner API, endpoints da Zenvia ou payload Meta Cloud API.

### Webhook de entrada

Endpoint do gateway: POST /webhooks/gupshup. Envelope mínimo ilustrativo:

```json
{
  "app": "Crefito11Homolog",
  "timestamp": 1790990000000,
  "version": 2,
  "type": "message",
  "payload": {
    "id": "wamid.exemplo-inbound",
    "source": "5561999999999",
    "type": "text",
    "payload": {"text": "Bom dia"},
    "sender": {"phone": "5561999999999", "name": "Contato de teste"}
  }
}
```

Mapear payload.id para inbound_whatsapp_id, payload.source para identidade de origem e payload.payload.text para texto. O número da empresa vem da configuração do app/canal, não de um campo to inventado. O timestamp do envelope é em milissegundos; converter explicitamente para UTC.

Responder 200/204 vazio somente após persistência. Payload inválido: 400. Origem não autorizada: 401/403. Falha de persistência: 503. Evento conhecido mas fora do escopo, como sandbox-start: registrar e confirmar sem criar mensagem. Eventos desconhecidos válidos: quarentena e alerta, sem transporte automático.

### Envio de texto

POST https://api.gupshup.io/wa/api/v1/msg

Header apikey; Content-Type application/x-www-form-urlencoded. Campos de formulário: channel=whatsapp, source, destination, src.name e message. Telefones são strings só com dígitos no transporte. message é uma string JSON serializada uma vez:

```json
{"type":"text","text":"Bom dia! Como posso ajudar?"}
```

O exemplo de sucesso da documentação retorna status=submitted e messageId. Salvar o ID imediatamente. Aceitação HTTP não comprova entrega; validar também corpo e status de erro. Aplicar percent-encoding via biblioteca HTTP, sem montar formulário por concatenação.

### Eventos de status

O envelope tem type=message-event; o status é payload.type.

| Evento | Correlação |
| --- | --- |
| enqueued | payload.id identifica Gupshup; payload.payload.whatsappMessageId permite registrar WhatsApp ID |
| sent / delivered / read | Preferir payload.gsId para Gupshup; payload.id representa WhatsApp ID |
| failed | Aceitar variante síncrona por id e variante assíncrona por gsId; guardar code/reason |

Não tratar payload.id como Gupshup ID em todos os eventos. Quando gsId faltar, buscar WhatsApp ID já mapeado. Eventos não correlacionados ficam pendentes. Converter payload.payload.ts, quando disponível, de segundos para UTC; guardar também o timestamp do envelope e received_at.

### Mídia e templates

Mídia inbound: extrair URL, validade, nome, legenda e MIME conforme payload homologado. file é o tipo de documento mostrado na referência V2. MIME declarado é metadado não confiável. E3 verificará cada schema outbound e limite vigente por tipo antes de implementar.

Templates: POST https://api.gupshup.io/wa/api/v1/template/msg, apikey e formulário com source, destination, src.name, channel e template contendo id e params. Não assumir o mesmo status/body de sucesso do endpoint de texto; a referência de template apresenta 202 com status=success e messageId. O adapter deve testar os dois contratos separadamente.

## Chatwoot 4.10.1

Usar Application API autenticada por header api_access_token. Prefixo B = /api/v1/accounts/{account_id}. inbox_identifier é próprio de Client API e não é necessário neste fluxo.

| Operação | Método e rota relativos a B |
| --- | --- |
| Criar contato | POST /contacts |
| Procurar contato | GET /contacts/search?q=...; confirmar correspondência exata e paginação |
| Ler contato | GET /contacts/{contact_id} |
| Criar vínculo | POST /contacts/{contact_id}/contact_inboxes |
| Conversas do contato | GET /contacts/{contact_id}/conversations |
| Criar conversa | POST /conversations |
| Ler conversa | GET /conversations/{conversation_id} |
| Abrir conversa | POST /conversations/{conversation_id}/toggle_status com status=open |
| Criar mensagem | POST /conversations/{conversation_id}/messages |
| Listar mensagens | GET /conversations/{conversation_id}/messages |
| Atualizar status | PATCH /conversations/{conversation_id}/messages/{message_id} |

As rotas e controllers da tag confirmam update de mensagem restrito a Inbox API. Os IDs de conversa nas rotas da conta são display_id; armazenar o ID exposto pela API, sem usar um ID interno obtido do banco.

### Contato e vínculo

Criar contato com name, phone_number em E.164 e identifier estável de integração. Antes de criar, procurar por telefone/identificador e validar igualdade; se houver ambiguidade, suspender vinculação e registrar para revisão. Não presumir que qualquer resultado de busca é a pessoa correta.

Criar vínculo com inbox_id e source_id estável. Salvar o valor retornado, inclusive quando o Chatwoot reutiliza um vínculo existente. Não fabricar um novo source_id em toda mensagem. Nunca sobrescrever nome/atributos de contato existente somente porque o perfil WhatsApp mudou.

### Conversa e mensagem

Criar conversa com source_id retornado pelo vínculo, inbox_id, contact_id e status=open. O request deve conter os três identificadores para evitar lookup apenas por source_id.

Incoming textual:

```json
{
  "content": "Bom dia",
  "message_type": "incoming",
  "private": false,
  "source_id": "gs-in:canal-teste:wamid.exemplo-inbound"
}
```

Anexos: multipart/form-data com attachments[], message_type=incoming, private=false e content opcional. Não enviar apenas URL em JSON esperando que o Chatwoot faça o download. O gateway baixa o arquivo de forma segura e envia bytes; validar MIME e limites em E3.

### Webhook de saída

Selecionar o webhook_url da própria Inbox API como mecanismo principal. Não configurar também webhook de conta para os mesmos eventos. A tag envia eventos da inbox por WebhookListener; a deduplicação continua obrigatória.

```json
{
  "event": "message_created",
  "id": 9845,
  "message_type": "outgoing",
  "private": false,
  "content": "Bom dia!",
  "account": {"id": 1},
  "inbox": {"id": 5},
  "conversation": {"id": 781}
}
```

Exemplo reduzido; payload real inclui mais campos. Validar conta/inbox, recuperar mensagem remotamente quando necessário e não confiar em telefone arbitrário do webhook. message_type no webhook da tag é string; representações numéricas de outros contextos não devem ser aceitas implicitamente.

### Status e falhas

PATCH da mensagem com {"status":"delivered"} ou {"status":"failed","external_error":"Falha de envio — referência técnica"}. A tag possui sent, delivered, read e failed; o gateway aplica regras próprias de não regressão. A proteção upstream contra regressão é parcial.

O status padrão da mensagem Chatwoot é sent. Ele não equivale a uma confirmação Gupshup. Enquanto o transporte está queued/submitted, acompanhar no gateway; não inventar status pending no Chatwoot. Quando bloqueado ou definitivamente falho, usar failed. Sanitizar external_error.

## Endpoints internos propostos

| Endpoint | Regra |
| --- | --- |
| POST /webhooks/gupshup | Validação, commit durável e ACK |
| POST /webhooks/chatwoot | Validação, commit durável e ACK |
| GET /health/live | Processo vivo; sem dados sensíveis |
| GET /health/ready | Banco e migrações disponíveis |
| POST /admin/conversations/{id}/templates | Autenticação administrativa, template da allowlist e Idempotency-Key obrigatório |
| POST /admin/jobs/{id}/reprocess | Autenticação, auditoria e rejeição de envio ambíguo sem reconciliação |

IDs administrativos são IDs do gateway. Endpoint de template aceita template_key e parâmetros definidos pelo catálogo local; resolve destinatário/canal pelo banco. Não aceita telefone livre. No MVP, a operação é técnica/autenticada; um seletor visual integrado ao Chatwoot é melhoria posterior.
