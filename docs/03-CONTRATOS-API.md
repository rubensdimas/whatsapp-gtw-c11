# Contratos das APIs

Links oficiais estão em [referências](09-REFERENCIAS.md). Exemplos são sintéticos. A homologação deve capturar os formatos reais da conta e da tag 4.10.1; não copiar payloads de outra geração de API.

## Gupshup: produto selecionado

Usar WhatsApp Self-Serve Messaging API, envio /wa/api/v1 e callback **V2**. A versão do envelope não é a versão do endpoint de envio. Não misturar com Partner API, endpoints da Zenvia ou payload Meta Cloud API.

### Webhook de entrada

Endpoint do gateway: POST /webhooks/gupshup/{channel_slug}/{secret} ([ADR 0005](adr/0005-segredo-na-rota-dos-webhooks.md)). O campo app deve coincidir com o provider_app do canal resolvido pelo slug. Envelope mínimo ilustrativo:

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

Responder 200/204 vazio somente após persistência. Payload inválido: 400. Slug ou segredo inválido: 404, sem criar evento. app divergente do canal: 403. Falha de persistência: 503. Evento conhecido mas fora do escopo, como sandbox-start: registrar e confirmar sem criar mensagem. Eventos desconhecidos válidos: quarentena e alerta, sem transporte automático.

### Envio de texto

POST https://api.gupshup.io/wa/api/v1/msg

Header apikey; Content-Type application/x-www-form-urlencoded. Campos de formulário: channel=whatsapp, source, destination, src.name e message. Telefones são strings só com dígitos no transporte. message é uma string JSON serializada uma vez:

```json
{"type":"text","text":"Bom dia! Como posso ajudar?"}
```

O exemplo de sucesso da documentação retorna status=submitted e messageId. Salvar o ID imediatamente. Aceitação HTTP não comprova entrega; validar também corpo e status de erro. Aplicar percent-encoding via biblioteca HTTP, sem montar formulário por concatenação.

### Formatação do texto de saída

Antes do envio, converter markdown do Chatwoot ([ADR 0009](adr/0009-formatacao-e-divisao-de-texto.md)):

| Chatwoot | WhatsApp |
| --- | --- |
| `**negrito**` | `*negrito*` |
| `_itálico_` / `*itálico*` | `_itálico_` |
| `~~riscado~~` | `~riscado~` |
| `` `código` `` e blocos | ```` ```código``` ```` |
| `[texto](url)` | `texto (url)` |
| Cabeçalhos, listas e citações | Texto simples, mantendo quebras de linha |

Texto acima de TEXT_MAX_CHARS (default 4096; confirmar limite vigente) vira partes ordenadas, quebradas por parágrafo, frase ou palavra, nessa preferência.

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

Mídia outbound: a URL entregue à Gupshup é `PUBLIC_BASE_URL/media/{token}`, servida pelo gateway a partir de cópia própria com validade MEDIA_URL_TTL_SECONDS ([ADR 0010](adr/0010-midia-de-saida-servida-pelo-gateway.md)). Nunca repassar URL de anexo do Chatwoot.

Resposta citada inbound: quando o payload trouxer contexto de resposta com ID de mensagem conhecida, prefixar o conteúdo com `↩︎ em resposta a: "<trecho>"`. O campo exato será confirmado nas fixtures; sem contexto reconhecido, enviar o conteúdo sem prefixo.

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

Criar contato com name, phone_number em E.164 e identifier `wa:<wa_id>`. Na 4.10.1, phone_number e identifier são únicos por conta; criar só depois de a resolução do [ADR 0002](adr/0002-resolucao-de-contato.md) não encontrar candidato. Busca retorna resultados aproximados: validar igualdade exata do campo antes de aceitar. Candidatos conflitantes entre passos: usar o mais forte, entregar a mensagem e criar aviso privado com os IDs para merge. Não presumir que qualquer resultado de busca é a pessoa correta.

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

Selecionar o webhook_url da própria Inbox API como mecanismo principal, apontando para a rede interna do Swarm: `http://<serviço-gateway>:<porta>/webhooks/chatwoot/{channel_slug}/{secret}`. Não configurar também webhook de conta para os mesmos eventos. A tag envia eventos da inbox por WebhookListener; a deduplicação continua obrigatória.

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

### Comandos em nota privada

Nota privada (private=true, outgoing, autor agente) na inbox do canal, cujo conteúdo começa com `/template` ou `/templates`, é comando ([ADR 0007](adr/0007-comando-de-template-em-nota-privada.md)). Demais notas são ignoradas.

| Comando | Efeito |
| --- | --- |
| `/templates` | Nota privada com chaves habilitadas e quantidade de parâmetros |
| `/template <chave> <p1> \| <p2> ...` | Valida catálogo e parâmetros, cria outgoing espelhado com texto renderizado e marcador gateway_template, agenda envio |

Parâmetros são separados por `|` e aparados. Erro de chave, parâmetros, contato sem telefone válido ou agente fora da allowlist do canal gera nota privada com o motivo, sem envio. Idempotência pelo ID da nota. Antes de implementar o filtro anti-loop, confirmar na Etapa 0 se content_attributes ou source_id do espelho aparece no webhook message_created da 4.10.1.

### Status e falhas

PATCH da mensagem com {"status":"delivered"} ou {"status":"failed","external_error":"Falha de envio — referência técnica"}. A tag possui sent, delivered, read e failed; o gateway aplica regras próprias de não regressão. A proteção upstream contra regressão é parcial.

O status padrão da mensagem Chatwoot é sent. Ele não equivale a uma confirmação Gupshup. Enquanto o transporte está queued/submitted, acompanhar no gateway; não inventar status pending no Chatwoot. Quando bloqueado ou definitivamente falho, usar failed. Sanitizar external_error.

## Endpoints internos propostos

| Endpoint | Regra |
| --- | --- |
| POST /webhooks/gupshup/{channel_slug}/{secret} | Publicado no Traefik; segredo, commit durável e ACK |
| POST /webhooks/chatwoot/{channel_slug}/{secret} | Somente rede interna; segredo, commit durável e ACK |
| GET /media/{token} | Publicado; arquivo temporário de saída enquanto válido (E3) |
| GET /health/live | Processo vivo; sem dados sensíveis |
| GET /health/ready | Banco e migrações disponíveis |
| POST /admin/channels/{slug}/conversations/{display_id}/templates | Token nomeado, template do catálogo e Idempotency-Key obrigatório |
| POST /admin/jobs/{id}/reprocess | Token nomeado, auditoria e rejeição de envio ambíguo sem reconciliação |

Endpoints /admin usam header `Authorization: Bearer <token>` com tokens nomeados por pessoa ([ADR 0006](adr/0006-tokens-administrativos-nomeados.md)). A rota de template usa o display_id visível no Chatwoot; IDs de job são do gateway. O endpoint aceita template_key e parâmetros do catálogo, resolve destinatário pelo banco e não aceita telefone livre.
