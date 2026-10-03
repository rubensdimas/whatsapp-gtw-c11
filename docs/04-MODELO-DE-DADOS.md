# Modelo de dados

Banco próprio PostgreSQL. Não compartilhar tabelas internas do Chatwoot. UUID nas entidades locais, bigint nos IDs Chatwoot, text nos IDs externos e timestamptz nos instantes. Telefones não são números inteiros. Payloads selecionados usam JSONB e retenção limitada.

## Entidades

| Entidade | Campos principais | Restrições |
| --- | --- | --- |
| channel | id, slug, provider, provider_app, source_phone, chatwoot_account_id, chatwoot_inbox_id, gupshup_secret_hashes, chatwoot_secret_hashes, template_agent_allowlist, enabled, paused_reason | Único slug; único account/inbox; único provider/provider_app; até dois hashes de segredo válidos por origem |
| channel_contact | id, channel_id, wa_id, phone_e164, chatwoot_contact_id, chatwoot_contact_inbox_source_id, last_user_message_at | Único channel_id/wa_id; único channel_id/source_id quando preenchido |
| channel_conversation | id, channel_contact_id, chatwoot_conversation_id, remote_status, is_current, last_checked_at | Único canal/account/conversation_id; um registro current por channel_contact |
| channel_message | id, channel_id, conversation_id, direction, inbound_whatsapp_id, chatwoot_message_id, content_kind, aggregate_status, failure_reason | Único channel/inbound ID; único channel/chatwoot_message_id |
| message_part | id, message_id, ordinal, kind, gupshup_message_id, whatsapp_message_id, transport_status, provider_event_at | Único message_id/ordinal; IDs externos únicos por canal quando preenchidos |
| send_attempt | id, message_part_id, ordinal, started_at, completed_at, result, response_code, provider_message_id | Único part/ordinal; preservar tentativa ambígua |
| webhook_event | id, channel_id, origin, event_key, payload, received_at, processing_status, processed_at | Único channel/origin/event_key |
| job | id, webhook_event_id, kind, payload_ref, state, attempts, next_attempt_at, locked_until, lock_owner, last_error | Índice state/next_attempt_at; unicidade do efeito planejado |
| status_receipt | id, channel_id, event_id, gupshup_id, whatsapp_id, status, provider_event_at, correlated_part_id | Evento imutável; permite correlação posterior |
| admin_command | id, channel_id, source, idempotency_key, request_hash, actor, command_type, state, result_ref | source rest ou note; único channel/source/idempotency_key; conflito se chave reutilizada com outro corpo |
| admin_token | id, name, token_hash, created_at, last_used_at, revoked_at | Único token_hash; token exibido uma vez |
| message_template | id, channel_id, key, provider_template_id, language, body, param_count, enabled | Único channel/key |
| media_object | id, message_part_id, storage_path, token_hash, mime, size_bytes, expires_at, deleted_at | Único token_hash; apagado após expirar (E3) |

Estes nomes são lógicos. A implementação deverá incluir FKs e o channel_id necessário a cada índice composto. Um índice unique parcial de is_current impede duas conversas correntes; isso é ponteiro local, não unicidade universal de conversa aberta no Chatwoot.

## Por que mais de três tabelas

As três entidades de contato/conversa/mensagem do desenho inicial não representam a recepção durável, retentativas, recibos fora de ordem ou anexos em múltiplas partes. As tabelas operacionais acima tornam essas garantias implementáveis; podem ser agrupadas fisicamente desde que suas invariantes sejam preservadas.

## Chaves de deduplicação

| Origem | Chave lógica |
| --- | --- |
| Incoming Gupshup | channel + message + payload.id |
| Outgoing Chatwoot | account + inbox + message_created + message.id |
| Recibo Gupshup | channel + tipo + IDs presentes + timestamp do evento + código de falha |
| Aviso privado técnico | message + categoria de erro |
| Comando template REST | channel + rest + Idempotency-Key |
| Comando template em nota | channel + note + ID da nota Chatwoot |
| Aviso de contato ambíguo | channel_contact + conjunto de candidatos |

Quando não houver identificador de evento, usar hash de JSON canônico dos campos relevantes. received_at não integra a chave. Recibos repetidos podem ser armazenados uma vez sem impedir eventos novos do mesmo ID.

## IDs distintos

- inbound_whatsapp_id: ID da mensagem recebida, vindo de payload.id.
- gupshup_message_id: messageId retornado no envio.
- whatsapp_message_id: ID de transporte, conhecido por enqueued ou recibos posteriores.
- chatwoot_message_id: ID da mensagem no painel.
- contact_inbox_source_id: identidade do vínculo; não é um ID de mensagem.
- chatwoot_conversation_id: ID público da conversa no escopo da conta.

Mesmo quando dois campos tiverem strings parecidas, não os fundir. Não correlacionar recibos por telefone e proximidade de horário.

## Estados locais

Job: ready → running → completed. Uma falha segura gera retry_wait; falha definitiva, dead; resultado externo incerto, reconciliation_required. Lease vencido permite reclaim de processamento interno, mas não autoriza repetir um envio externo iniciado.

Transporte por parte: queued → submitting → submitted → enqueued → sent → delivered → read. failed e reconciliation_required são ramos, não degraus de uma ordenação linear. Um recibo pode saltar etapas. Uma parte em reconciliation_required guarda hold_until; vencido o prazo, a mensagem recebe failed no Chatwoot e libera a fila do contato, mas a parte continua reconciliation_required até decisão da TI ([ADR 0003](adr/0003-retencao-da-fila-por-contato.md)).

Status agregado de mensagem com várias partes: failed se uma parte tiver falha definitiva; read somente se todas estiverem read; delivered se todas estiverem delivered/read; sent se todas estiverem sent/delivered/read. Antes disso, permanecer submitted/queued localmente. Preservar partes entregues mesmo quando outra falhar.

## Janela e concorrência

last_user_message_at recebe o maior instante válido de incoming do usuário, sem regressão por evento atrasado. Rejeitar/quarentenar timestamps futuros além da tolerância configurada. A validade de envio livre é last_user_message_at + 24 horas, reavaliada no momento de cada tentativa.

Serializar criação de vínculo/conversa por channel_contact. Aplicar unicidade e conflito transacional como última defesa; locks locais de processo não bastam com múltiplos workers.

## Retenção proposta

Payloads brutos: 7 dias; metadados de correlação/recibos: 90 dias. O expurgo de payload apaga o conteúdo JSONB e mantém a linha e event_key, para que reentregas tardias continuem deduplicadas. Mídia temporária segue MEDIA_URL_TTL_SECONDS. São defaults técnicos para avaliação do Conselho, não uma tabela de temporalidade legal. No piloto usar dados de teste. Não aplicar expurgo em produção antes de validar a política institucional e as necessidades de auditoria. Backups devem respeitar o mesmo controle de acesso.
