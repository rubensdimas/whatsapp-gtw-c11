# Configuração e operação

## Pré-requisitos

App Gupshup Self-Serve, número de teste habilitado, chave de API, callback V2 e assinatura dos eventos message e message-event. Chatwoot 4.10.1 com Inbox API dedicada, atendentes associados e token técnico. HTTPS público para callbacks, PostgreSQL próprio e rede para APIs externas.

Não presumir que o número atual pode operar simultaneamente em dois conectores. Migração, coexistência, faturamento e habilitação do número são atividades separadas, dependentes do provedor e de autorização específica.

## Preparação passo a passo

Antes da homologação no Conselho, o desenvolvimento usa o compose local com Chatwoot 4.10.1 e o sandbox Gupshup por túnel HTTPS temporário ([ADR 0012](adr/0012-etapa-1-em-paralelo.md)). O túnel é desligado após cada captura e nunca recebe tráfego de produção.

1. Criar Inbox API de homologação chamada WhatsApp — Gupshup Homologação. Anotar account_id e inbox_id; associar equipe de teste.
2. Criar usuário técnico/token e testar acesso restrito às operações necessárias. Não compartilhar token pessoal do administrador nos ambientes.
3. Vincular o número de teste ao app Gupshup existente; confirmar que o produto contratado corresponde aos contratos documentados.
4. Publicar gateway de homologação em HTTPS. Cadastrar o canal com `gateway channels add` e gerar segredos com `gateway channels rotate-secret`. Só cadastrar o callback depois que persistência e ACK estiverem funcionando.
5. Configurar callback Gupshup V2 com a URL contendo slug e segredo; habilitar inbound e recibos. Capturar eventos reais anonimizados para fixtures.
6. Configurar webhook_url da Inbox API com a URL interna do Swarm. Evitar webhook de conta adicional para o mesmo fluxo.
7. Validar contato, vínculo/source_id, conversa, incoming e PATCH de status na instalação 4.10.1.
8. Criar tokens administrativos para cada pessoa da TI com `gateway admin-tokens create`.
9. Iniciar somente E1 em piloto; habilitar E2 e E3 após seus respectivos testes.

## Configuração proposta

Nomes abaixo pertencem ao gateway; não são nomes oficiais de variáveis do Chatwoot ou Gupshup. Nenhum segredo real está incluído. Dados de canal (app, número, conta, inbox, segredos de rota) ficam na tabela channel e são geridos pela CLI ([ADR 0001](adr/0001-canais-no-banco.md)); não há variável por canal exceto a chave Gupshup.

| Variável | Uso |
| --- | --- |
| APP_ENV | local, homolog ou production |
| PUBLIC_BASE_URL | Base HTTPS do gateway (rotas Gupshup e /media) |
| DATABASE_URL | PostgreSQL próprio; credencial de serviço |
| CHATWOOT_BASE_URL | URL da instalação, preferencialmente interna ao Swarm |
| CHATWOOT_API_TOKEN | Header api_access_token do usuário técnico |
| CHATWOOT_EXPECTED_VERSION | 4.10.1; referência de homologação |
| GUPSHUP_API_KEY__<SLUG> | Header apikey do canal; SLUG em maiúsculas |
| GUPSHUP_API_BASE_URL | https://api.gupshup.io |
| HTTP_CONNECT_TIMEOUT_SECONDS / HTTP_READ_TIMEOUT_SECONDS | Timeouts de cliente |
| JOB_MAX_ATTEMPTS / JOB_LEASE_SECONDS | Retry seguro e lease |
| ORDERING_HOLD_SECONDS | Retenção da fila do contato após envio ambíguo; default 300 |
| TEXT_MAX_CHARS | Limite por parte de texto; default 4096 |
| MEDIA_MAX_BYTES / MEDIA_ALLOWED_HOSTS | Política local dentro dos limites de cada API |
| MEDIA_STORAGE_PATH / MEDIA_URL_TTL_SECONDS | Volume e validade da mídia de saída; default 3600 |
| RAW_EVENT_RETENTION_DAYS / METADATA_RETENTION_DAYS | Retenção técnica aprovada |
| TEMPLATES_ENABLED / MEDIA_ENABLED / OUTBOUND_ENABLED | Flags de habilitação por etapa |

## CLI de operação

Executada dentro do container, com acesso ao banco. Saída nunca imprime segredos já gravados.

| Comando | Uso |
| --- | --- |
| `gateway channels add / list / pause / resume` | Cadastro e estado dos canais |
| `gateway channels rotate-secret <slug> --origin gupshup\|chatwoot` | Gera segredo novo, mantém o anterior até `--finish` |
| `gateway admin-tokens create --name / list / revoke` | Tokens nomeados da TI |
| `gateway templates add / disable / list` | Catálogo de templates por canal |

## Deployment proposto

Imagem única com processos separados gateway-api e gateway-worker. Banco e volume de mídia (MEDIA_STORAGE_PATH) duráveis, montado na API e no worker. Migrações executadas uma vez antes de iniciar workers; não por toda réplica ao subir. Compose para desenvolvimento, com perfil opcional Chatwoot 4.10.1. Swarm/Traefik no mesmo cluster do Chatwoot: gateway-api na rede overlay do Chatwoot; Traefik publica somente /webhooks/gupshup/*, /media/* e /admin/* (este último restrito à rede da TI).

Uma réplica API e um worker bastam ao piloto; dimensionar por métricas. Health/readiness da API avalia dependências locais; worker expõe heartbeat e tarefas em atraso. Não expor PostgreSQL ao público. Não há configuração funcional de Compose/stack neste pacote.

## Template no MVP

Catálogo local de templates textuais aprovados, gerido pela TI com `gateway templates`: chave interna, ID Gupshup, idioma, texto com placeholders, quantidade de parâmetros e disponibilidade ([ADR 0007](adr/0007-comando-de-template-em-nota-privada.md)).

Atendentes usam nota privada na conversa: `/templates` lista as chaves; `/template <chave> <p1> | <p2>` envia. A TI pode usar o endpoint administrativo com display_id e Idempotency-Key. Registrar ator (autor da nota ou token), parâmetros mínimos necessários e ID de envio.

O comando cria uma mensagem outgoing espelhada no Chatwoot com origem gateway_template e marcador de não retransmissão, antes de executar a saída pelo job. O filtro desse marcador precisa de teste contra loop e abuso. Um dashboard visual para atendentes fica para melhoria posterior.

## Responsabilidade operacional

A TI do Conselho recebe alertas, decide reconciliações (reconciliation_required), reprocessa jobs, pausa canais, rotaciona segredos e mantém o catálogo de templates. Atendentes tratam avisos privados (janela, contato ambíguo, envio incerto) no próprio Chatwoot.

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
| Resultado incerto | Fila do contato liberada após ORDERING_HOLD_SECONDS; TI inspeciona tentativa/recibos; não reenviar automaticamente |
| Segredo de rota vazado | `rotate-secret`, atualizar callback/webhook_url, concluir rotação com `--finish` |
| Erro de contrato após upgrade | Pausar capacidade afetada; capturar fixture e revisar contrato |

Reprocessamento de transporte e de atualização de status são ações distintas. Fazer backup antes de migrações e testar restauração no ambiente de teste. No rollback, pausar workers/saída, preservar banco e eventos e retomar com versão compatível; não apagar mapeamentos.
