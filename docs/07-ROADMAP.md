# Roadmap de implementação

Backlog inicial para transformar os documentos em um plano de código. A aplicação ainda não foi implementada. Estimativas só devem ser produzidas após a etapa 0, quando os contratos reais estiverem capturados.

## Etapa 0 — validar contratos e preparar ambiente

- Confirmar app/produto Gupshup, número autorizado e callback V2.
- Criar Inbox API em Chatwoot 4.10.1 e validar permissões do token.
- Capturar fixtures: incoming texto, outgoing público, nota privada, enqueued, sent, delivered, read e failed.
- Testar create contact, contact_inbox, create conversation, incoming e PATCH de status.
- Verificar autenticação dos callbacks e formatos reais de erros.

Saída: matriz de contratos preenchida com evidências anonimizadas. Impedimentos de acesso são bloqueios externos, não motivo para inventar comportamento.

## Etapa 1 — fundação

- Estruturar FastAPI e worker com configuração validada.
- Fixar versões Python/dependências e preparar cliente HTTP com timeouts.
- Criar migrações, unicidades, webhook_event, jobs e estados de tentativa.
- Implementar recepção com commit antes do ACK, health e filtros de origem.
- Testar reinicialização, duplicação e indisponibilidade do banco.

Saída: eventos duráveis e recuperáveis, sem chamadas externas dentro do request de webhook.

## Etapa 2 — texto inbound

- Implementar normalização Gupshup e identidade por canal.
- Resolver contato existente, vínculo e conversa com concorrência controlada.
- Inserir incoming com source_id externo e registrar correlação.
- Reconciliar criação Chatwoot com retorno perdido.
- Abrir atendimento reutilizado e criar nova conversa após resolved.

Saída: mensagem recebida aparece uma vez em situações conhecidas, inclusive após reentrega de webhook e reinício.

## Etapa 3 — texto outbound e piloto E1

- Filtrar conta/inbox, message_created, outgoing e private=false.
- Recuperar destinatário pelo mapeamento, validar janela e registrar tentativa.
- Implementar formulário de envio Gupshup e leitura do corpo de resposta.
- Tratar erro definitivo, ACK perdido e aviso privado sem loop.
- Validar ordenação por contato e deduplicação por ID Chatwoot.

Saída: piloto textual com atendimento humano e bloqueio fora da janela. Ainda sem promessa de sincronização completa de recibos.

## Etapa 4 — status e operação E2

- Distinguir messageId, gsId, WhatsApp ID e Chatwoot ID.
- Implementar recibos pendentes, progressão e agregação.
- Sincronizar PATCH sent/delivered/read/failed em 4.10.1.
- Separar retry de status de retry de transporte.
- Adicionar métricas, alertas, inspeção e reprocessamento auditado.

Saída: textos com recibos e falhas visíveis; indisponibilidade e eventos fora de ordem homologados.

## Etapa 5 — mídia e templates E3

- Confirmar schemas e limites vigentes outbound para imagem, áudio, vídeo e arquivo.
- Baixar inbound com proteção SSRF e enviar multipart para Chatwoot.
- Disponibilizar mídia de saída por URL temporária compatível com o provedor.
- Implementar partes, legendas, ordem e agregado de status.
- Criar catálogo de template textual e endpoint administrativo autenticado/idempotente.
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
