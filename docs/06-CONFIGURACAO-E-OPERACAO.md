# Configuração e operação

## Pré-requisitos

App Gupshup Self-Serve, número de teste habilitado, chave de API, callback V2 e assinatura dos eventos message e message-event. Chatwoot 4.10.1 com Inbox API dedicada, atendentes associados e token técnico. HTTPS público para callbacks, PostgreSQL próprio e rede para APIs externas.

Não presumir que o número atual pode operar simultaneamente em dois conectores. Migração, coexistência, faturamento e habilitação do número são atividades separadas, dependentes do provedor e de autorização específica.

## Preparação passo a passo

1. Criar Inbox API de homologação chamada WhatsApp — Gupshup Homologação. Anotar account_id e inbox_id; associar equipe de teste.
2. Criar usuário técnico/token e testar acesso restrito às operações necessárias. Não compartilhar token pessoal do administrador nos ambientes.
3. Preparar app/número Gupshup de teste; confirmar que o produto contratado corresponde aos contratos documentados.
4. Publicar gateway de homologação em HTTPS e configurar validação de origem. Só cadastrar o callback depois que persistência e ACK estiverem funcionando.
5. Configurar callback Gupshup V2; habilitar inbound e recibos. Capturar eventos reais anonimizados para fixtures.
6. Configurar webhook_url da Inbox API para o gateway. Evitar webhook de conta adicional para o mesmo fluxo.
7. Validar contato, vínculo/source_id, conversa, incoming e PATCH de status na instalação 4.10.1.
8. Iniciar somente E1 em piloto; habilitar E2 e E3 após seus respectivos testes.

## Configuração proposta

Nomes abaixo pertencem ao gateway; não são nomes oficiais de variáveis do Chatwoot ou Gupshup. Nenhum segredo real está incluído.

| Variável | Uso |
| --- | --- |
| APP_ENV | local, homolog ou production |
| PUBLIC_BASE_URL | Base HTTPS do gateway |
| DATABASE_URL | PostgreSQL próprio; credencial de serviço |
| CHATWOOT_BASE_URL | URL real da instalação |
| CHATWOOT_ACCOUNT_ID / CHATWOOT_INBOX_ID | Escopo autorizado |
| CHATWOOT_API_TOKEN | Header api_access_token |
| CHATWOOT_EXPECTED_VERSION | 4.10.1; referência de homologação |
| GUPSHUP_API_KEY | Header apikey |
| GUPSHUP_APP_NAME | App cadastrado, usado em src.name e filtro inbound |
| GUPSHUP_SOURCE | Número empresarial, dígitos com DDI |
| GUPSHUP_API_BASE_URL | https://api.gupshup.io |
| GUPSHUP_WEBHOOK_SECRET / CHATWOOT_WEBHOOK_SECRET | Segredos locais do callback, conforme estratégia homologada |
| ADMIN_API_TOKEN | Credencial exclusiva dos endpoints administrativos |
| HTTP_CONNECT_TIMEOUT_SECONDS / HTTP_READ_TIMEOUT_SECONDS | Timeouts de cliente |
| JOB_MAX_ATTEMPTS / JOB_LEASE_SECONDS | Retry seguro e lease |
| MEDIA_MAX_BYTES / MEDIA_ALLOWED_HOSTS | Política local dentro dos limites de cada API |
| RAW_EVENT_RETENTION_DAYS / METADATA_RETENTION_DAYS | Retenção técnica aprovada |
| TEMPLATES_ENABLED / MEDIA_ENABLED / OUTBOUND_ENABLED | Flags de habilitação por etapa |

## Deployment proposto

Imagem única com processos separados gateway-api e gateway-worker. Banco e storage com volumes duráveis. Migrações executadas uma vez antes de iniciar workers; não por toda réplica ao subir. Compose para desenvolvimento; Swarm/Traefik para o ambiente do Conselho se confirmado.

Uma réplica API e um worker bastam ao piloto; dimensionar por métricas. Health/readiness da API avalia dependências locais; worker expõe heartbeat e tarefas em atraso. Não expor PostgreSQL ao público. Não há configuração funcional de Compose/stack neste pacote.

## Template no MVP

Manter catálogo local de templates textuais aprovados: chave interna, ID Gupshup, idioma, parâmetros e status de disponibilidade. Envio pelo endpoint administrativo autenticado, com destinatário derivado da conversa e Idempotency-Key. Registrar ator, parâmetros mínimos necessários e ID de envio.

O comando cria/associa uma mensagem outgoing no Chatwoot com origem gateway_template e marcador de não retransmissão, antes de executar a saída pelo job. O filtro desse marcador precisa de teste contra loop e abuso. A API Inbox não fornece automaticamente o seletor de templates de uma Inbox WhatsApp nativa. Um dashboard visual para atendentes fica para melhoria posterior.

## Observabilidade

Medir ACK p95, latência de incoming/outgoing, jobs por estado, idade da fila, retries, mensagens com resultado incerto, recibos pendentes, erros por provedor e worker heartbeat. Logs estruturados com correlation_id, channel_id, message_id e categoria; sem corpo do atendimento.

Defaults de alerta propostos: worker sem heartbeat > 60 s; job mais antigo > 5 min; qualquer 401/403; taxa de falhas > 5% em 10 min com pelo menos 20 tentativas; qualquer resultado incerto sem ação > 15 min.

## Recuperação

| Ocorrência | Ação |
| --- | --- |
| Chatwoot indisponível | Guardar jobs; retomar chamadas seguras após recuperação |
| Gupshup indisponível | Pausar saída, preservar ordem e revalidar janela quando retomar |
| Banco indisponível | Responder 503 em callbacks; recuperar banco antes do ACK |
| Credencial inválida | Pausar canal, substituir secret, validar e liberar |
| Resultado incerto | Inspecionar tentativa/recibos; não reenviar automaticamente |
| Erro de contrato após upgrade | Pausar capacidade afetada; capturar fixture e revisar contrato |

Reprocessamento de transporte e de atualização de status são ações distintas. Fazer backup antes de migrações e testar restauração no ambiente de teste. No rollback, pausar workers/saída, preservar banco e eventos e retomar com versão compatível; não apagar mapeamentos.
