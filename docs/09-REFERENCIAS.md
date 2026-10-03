# Referências oficiais e compatibilidade

Consulta realizada em 03/10/2026. A documentação online é evolutiva; a versão usada pelo Conselho é **Chatwoot 4.10.1**. Referências e contratos devem ser revisitados durante a implementação, especialmente mídia, limites, callback e regras de envio.

## Gupshup

| Fonte | Uso |
| --- | --- |
| [Índice da documentação](https://docs.gupshup.io/llms.txt) | Descoberta dos contratos oficiais |
| [Texto inbound V2](https://docs.gupshup.io/docs/text) | Estrutura de message e conteúdo textual |
| [Webhook Key Points](https://docs.gupshup.io/docs/what-is-a-webhook) | ACK e processamento assíncrono |
| [Message events V2](https://docs.gupshup.io/docs/message-events) | gsId, WhatsApp ID, recibos e timestamps |
| [Envio de texto](https://docs.gupshup.io/reference/session-text-message) | /wa/api/v1/msg, formulário, apikey e retorno |
| [Mídia inbound](https://docs.gupshup.io/docs/media) | URL, validade e metadados de mídia |
| [Template textual](https://docs.gupshup.io/reference/sending-text-template) | /wa/api/v1/template/msg e parâmetros |

As páginas acima foram abertas nesta preparação. Schemas outbound de cada mídia e limites atuais devem ser consultados no índice e homologados em E3. Não foram fixados tamanhos nem formatos não verificados.

## Chatwoot — documentação oficial

| Fonte | Uso |
| --- | --- |
| [Índice](https://developers.chatwoot.com/llms.txt) | Navegação das APIs |
| [Criar contato](https://developers.chatwoot.com/api-reference/contacts/create-contact) | Dados de contato |
| [Criar contact inbox](https://developers.chatwoot.com/api-reference/contacts/create-contact-inbox) | Vínculo e source_id |
| [Criar conversa](https://developers.chatwoot.com/api-reference/conversations/create-new-conversation) | Conversa em Inbox API |
| [Criar mensagem](https://developers.chatwoot.com/api-reference/messages/create-new-message) | Texto e anexos |
| [Criar webhook de conta](https://developers.chatwoot.com/api-reference/webhooks/add-a-webhook) | Alternativa ao callback da inbox, não usar ambos para o mesmo fluxo |
| [Atualizar status](https://developers.chatwoot.com/api-reference/messages/update-message-status) | Referência listada no índice; conteúdo da página não pôde ser recuperado nesta preparação |

O contrato de atualização de status deste pacote foi confirmado pelo código da tag, sem depender da página acima.

## Chatwoot — código fixado em v4.10.1

Estes arquivos foram lidos diretamente do repositório oficial pela URL raw da tag; os links abaixo permitem a revisão humana no GitHub.

| Arquivo | Constatação |
| --- | --- |
| [config/routes.rb](https://github.com/chatwoot/chatwoot/blob/v4.10.1/config/routes.rb) | Recurso messages inclui update nas rotas da conta/conversa |
| [MessagesController](https://github.com/chatwoot/chatwoot/blob/v4.10.1/app/controllers/api/v1/accounts/conversations/messages_controller.rb) | update chama serviço de status; exige Inbox API e aceita external_error |
| [StatusUpdateService](https://github.com/chatwoot/chatwoot/blob/v4.10.1/app/services/messages/status_update_service.rb) | Valida enum e bloqueia parcialmente regressão read → delivered |
| [Message](https://github.com/chatwoot/chatwoot/blob/v4.10.1/app/models/message.rb) | Enum sent/delivered/read/failed; sent padrão; webhook com message_type string |
| [MessageBuilder](https://github.com/chatwoot/chatwoot/blob/v4.10.1/app/builders/messages/message_builder.rb) | Incoming restrito a Inbox API, source_id e anexos |
| [ContactInboxesController](https://github.com/chatwoot/chatwoot/blob/v4.10.1/app/controllers/api/v1/accounts/contacts/contact_inboxes_controller.rb) | Criação de vínculo por inbox_id/source_id |
| [ConversationsController](https://github.com/chatwoot/chatwoot/blob/v4.10.1/app/controllers/api/v1/accounts/conversations_controller.rb) | Rotas de conta localizam conversa por display_id |
| [Channel::Api](https://github.com/chatwoot/chatwoot/blob/v4.10.1/app/models/channel/api.rb) | API Inbox possui webhook_url |
| [WebhookListener](https://github.com/chatwoot/chatwoot/blob/v4.10.1/app/listeners/webhook_listener.rb) | Webhooks de inbox e conta são caminhos distintos; ambos podem emitir o mesmo evento |
| [Contact](https://github.com/chatwoot/chatwoot/blob/v4.10.1/app/models/contact.rb) | phone_number (E.164) e identifier únicos por conta; base do [ADR 0002](adr/0002-resolucao-de-contato.md) |

## Grau de confirmação

| Item | Estado |
| --- | --- |
| Estrutura V2/texto/status Gupshup | Confirmada nas referências consultadas |
| PATCH de status em Inbox API Chatwoot 4.10.1 | Confirmado no código upstream |
| Configuração e permissões da instalação do Conselho | Ainda não testadas |
| Assinatura/autenticação real dos callbacks | Validar na conta/rede de homologação |
| Reuso/reabertura de conversa, busca e resposta JSON exata | Validar com fixtures e testes 4.10.1 |
| Schema/limites outbound de mídia | Validar em E3 |
| Unicidade de phone_number/identifier por conta | Confirmada no código upstream |
| content_attributes/source_id no webhook message_created | Validar na Etapa 0 |
| Transição failed → delivered aceita pelo PATCH | Validar na Etapa 0 |
| Formato V2 do sandbox igual ao do número vinculado | Validar ao vincular o número |
| Campo de contexto de resposta citada inbound | Validar com fixtures |
| Limite de caracteres de texto | Confirmar na referência vigente |
| Migração/coexistência do número de produção | Fora deste pacote |

## Decisões de projeto

Fila PostgreSQL, janela calculada localmente, contratos internos, endpoints administrativos, métricas, retenção e políticas de retry são decisões propostas para o gateway. As decisões tomadas após a preparação estão em [docs/adr](adr/README.md). Não são recursos ou garantias atribuídos à Gupshup ou ao Chatwoot. A arquitetura aprovada pelo usuário foi preservada; as definições adicionais tornam o trabalho implementável e passível de revisão.
