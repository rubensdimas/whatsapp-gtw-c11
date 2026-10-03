# Roadmap de implementação

Backlog inicial para transformar os documentos em um plano de código. A aplicação ainda não foi implementada. Estimativas só devem ser produzidas após a etapa 0, quando os contratos reais estiverem capturados.

A Etapa 0 corre em paralelo às Etapas 1 e 2 ([ADR 0012](adr/0012-etapa-1-em-paralelo.md)). Situação em 03/10/2026: app Gupshup criado, número de teste ainda não vinculado; Inbox e Swarm de homologação não confirmados.

## Etapa 0 — validar contratos e preparar ambiente

- Vincular o número de teste ao app Gupshup existente e confirmar produto e callback V2.
- Capturar payloads do sandbox Gupshup por túnel HTTPS temporário, logo após a Etapa 1; reconferir depois com o número vinculado.
- Subir Chatwoot 4.10.1 no compose local; depois criar Inbox API de homologação no Conselho e validar permissões do token.
- Capturar fixtures: incoming texto, resposta citada, outgoing público, nota privada, enqueued, sent, delivered, read e failed.
- Testar create contact, contact_inbox, create conversation, incoming e PATCH de status, inclusive failed → delivered.
- Verificar se content_attributes e source_id de mensagem criada via API aparecem no webhook message_created (filtro do espelho de template).
- Verificar formatos reais de erros e se a Gupshup oferece autenticação melhor que segredo na rota.

Saída: matriz de contratos preenchida com evidências anonimizadas. Impedimentos de acesso são bloqueios externos, não motivo para inventar comportamento.

## Etapa 1 — fundação

- Estruturar projeto com uv, ruff, mypy, pytest/testcontainers e CI ([ADR 0011](adr/0011-tooling.md)).
- Estruturar FastAPI e worker com configuração validada.
- Preparar cliente HTTP com timeouts.
- Criar migrações: channel, webhook_event, job, admin_token e estados de tentativa, com unicidades.
- CLI de canais, rotação de segredos e tokens administrativos.
- Implementar rotas com slug/segredo, commit antes do ACK, mascaramento de logs e health.
- Implementar claim/lease do worker e retenção da fila por contato.
- Testar reinicialização, duplicação e indisponibilidade do banco.

Saída: eventos duráveis e recuperáveis, sem chamadas externas dentro do request de webhook.

## Etapa 2 — texto inbound

- Implementar normalização Gupshup e identidade por canal, inicialmente com fixtures provisórias.
- Resolver contato por mapeamento, identifier, telefone exato e variante do nono dígito; aviso de ambiguidade ([ADR 0002](adr/0002-resolucao-de-contato.md)).
- Resolver vínculo e conversa com concorrência controlada.
- Prefixar resposta citada quando a original for conhecida.
- Inserir incoming com source_id externo e registrar correlação.
- Reconciliar criação Chatwoot com retorno perdido.
- Abrir atendimento reutilizado e criar nova conversa após resolved.

Saída: mensagem recebida aparece uma vez em situações conhecidas, inclusive após reentrega de webhook e reinício.

## Etapa 3 — texto outbound e piloto E1

- Filtrar conta/inbox, message_created, outgoing e private=false.
- Recuperar destinatário pelo mapeamento ou, em conversa iniciada pelo atendente, pelo telefone do contato ([ADR 0004](adr/0004-conversa-iniciada-pelo-atendente.md)); validar janela e registrar tentativa.
- Converter markdown e dividir texto longo em partes ([ADR 0009](adr/0009-formatacao-e-divisao-de-texto.md)).
- Implementar formulário de envio Gupshup e leitura do corpo de resposta.
- Implementar PATCH failed e aviso privado deduplicado ([ADR 0008](adr/0008-status-failed-no-e1.md)).
- Tratar erro definitivo, ACK perdido, expiração da retenção da fila e aviso privado sem loop.
- Validar ordenação por contato e deduplicação por ID Chatwoot.

Saída: piloto textual com atendimento humano, bloqueio fora da janela e falhas visíveis. Ainda sem promessa de sincronização de recibos positivos.

## Etapa 4 — status e operação E2

- Distinguir messageId, gsId, WhatsApp ID e Chatwoot ID.
- Implementar recibos pendentes, progressão e agregação.
- Sincronizar PATCH sent/delivered/read em 4.10.1, somando-se ao failed do E1.
- Separar retry de status de retry de transporte.
- Adicionar métricas, alertas, inspeção e reprocessamento auditado.

Saída: textos com recibos e falhas visíveis; indisponibilidade e eventos fora de ordem homologados.

## Etapa 5 — mídia e templates E3

- Confirmar schemas e limites vigentes outbound para imagem, áudio, vídeo e arquivo.
- Baixar inbound com proteção SSRF e enviar multipart para Chatwoot.
- Disponibilizar mídia de saída em volume próprio por /media/{token} com TTL e limpeza ([ADR 0010](adr/0010-midia-de-saida-servida-pelo-gateway.md)).
- Implementar partes, legendas, ordem e agregado de status.
- Criar catálogo de template (CLI), comandos `/templates` e `/template` em nota privada e endpoint administrativo por display_id ([ADR 0007](adr/0007-comando-de-template-em-nota-privada.md)).
- Espelhar template no histórico e verificar filtro contra retransmissão.

Saída: todos os tipos previstos e template aprovado homologados. Recursos não suportados produzem aviso técnico explícito.

## Etapa 6 — liberação

- Executar matriz de homologação e coletar evidências.
- Validar retenção institucional, autenticação de callbacks, backup/restauração e monitoração.
- Revisar rollback e capacitar equipe sobre janela, templates e falhas.
- Definir mudança do número/fornecedor em processo separado, se necessária.

Saída: MVP operacional pronto para autorização de entrada em produção. Estes arquivos não executam a mudança.

## Próxima entrega

IA só será planejada após estabilidade do transporte. O adapter e as entidades do gateway preservam a possibilidade futura, sem criar agora roteador IA, autenticação PF/PJ, RAG ou conexão com Implanta.NET.
