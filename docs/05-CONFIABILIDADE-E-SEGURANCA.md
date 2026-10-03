# Confiabilidade e segurança

## Recepção durável

Autenticar/validar → transação que cria evento e job → commit → ACK vazio. Evento duplicado recebe ACK após comprovar que já está persistido. Se o banco falhar, devolver 503. Processar externamente só no worker. A Gupshup documenta repetição quando não recebe ACK em dez segundos; não usar esse prazo como meta interna.

## Política de retentativas

| Situação | Tratamento |
| --- | --- |
| GET falha transitoriamente | Retry com backoff e jitter |
| 429 que confirma rejeição sem processamento | Respeitar Retry-After quando presente; retry limitado |
| 400/422 de contrato | Falha definitiva e alerta; corrigir dados antes de reprocessar |
| 401/403 | Pausar canal e alertar responsável técnico |
| Timeout/reset após POST iniciado | reconciliation_required; sem repetição cega |
| 5xx após POST | Resultado potencialmente incerto; reconciliar, salvo garantia explícita de rejeição |
| PATCH de status com falha transitória | Retry do status, sem reenviar conteúdo |
| URL de mídia expirada | Falha de mídia rastreável; não repetir download indefinidamente |

Defaults propostos: timeout de conexão 3 s, leitura 15 s para texto; 60 s para mídia. Até cinco tentativas seguras com atrasos base de 2, 10, 30, 120 e 600 s e jitter. Após esgotamento, dead e alerta. Definir os valores no código/configuração e testar; nenhum deles é limite oficial dos provedores.

## Limite da deduplicação

Persistência local impede repetir um efeito já conhecido. Não existe transação distribuída entre gateway, Gupshup e Chatwoot. Se um POST concluir externamente e sua resposta se perder, o gateway não pode presumir que falhou.

Para Chatwoot, reconciliar criação de contato/vínculo/conversa por identidade exata e mensagem incoming por source_id externo ao listar mensagens paginadas. Para Gupshup, aguardar recibos e usar consultas oficiais apenas se houver contrato disponível e homologado. Não inventar endpoint de consulta por chave idempotente. Sem prova suficiente, intervenção técnica decide se há reenvio, registrando o risco.

Recibos que chegam antes de messageId ser salvo permanecem pendentes e são reavaliados quando o mapeamento for gravado. Reconciliar periodicamente; após 24 h sem associação, alertar e continuar retendo conforme política.

## Progressão de status

Aplicar sent < delivered < read para recibos positivos. Um sent atrasado não rebaixa read. A falha não pertence a essa sequência; preservar histórico e conferir tentativa/IDs. Um failed contraditório depois de delivered/read exige revisão, sem rebaixamento automático. Não inferir falha apenas porque read não chegou.

Para múltiplas partes, sincronizar o agregado. Uma nova tentativa autorizada fica identificada separadamente; recibos antigos não substituem os da tentativa corrente. O serviço de status upstream tem proteção parcial; o gateway fornece a regra completa.

## Prevenção de loops

Transportar somente message_created, outgoing, private=false e conta/inbox permitidas. Ignorar message_updated e notas privadas. Um aviso privado do gateway nunca chega ao WhatsApp. Se uma mensagem de template for espelhada como outgoing pelo próprio gateway, registrar antes sua origem/comando em content_attributes e aplicar filtro verificado; deduplicação por ID completa a defesa.

## Janela de atendimento

Usar a última mensagem válida enviada pelo usuário, não a data de criação da conversa, resolved/open, último envio do atendente ou conversation.expiresAt de cobrança. Texto livre fora de 24 h é bloqueado antes da API; atualizar failed e informar privadamente. Enviar template não reabre sozinho a janela de texto livre: aguardar nova mensagem do usuário.

Esta é uma restrição operacional de envio; não implica mensagem gratuita nem estimativa de preço. Revalidar regras oficiais durante a implementação.

## Proteção dos webhooks

Não assumir que apikey de envio autentica callback. As referências consultadas não estabelecem uma assinatura HMAC universal para este fluxo. Validar suporte real de autenticação/assinatura na conta antes de implementar header ou algoritmo.

No piloto, restringir origem no proxy e usar URL com segredo aleatório de alta entropia se o callback não oferecer mecanismo melhor; omitir segredo de access logs, manter rotacionável e não publicá-lo. Para Gupshup, obter lista oficial de IPs via suporte; não inventar ranges. Para Chatwoot no mesmo ambiente, restringir a rede/IP de origem e, quando houver acesso pelo endpoint público, conferir o objeto pela Application API antes de enviar.

app, account_id e inbox_id são filtros de escopo, não prova de autenticidade. HMAC do vínculo contact_inbox não deve ser confundido com assinatura do webhook do Chatwoot. Bloquear produção até validar o mecanismo aplicável a cada origem.

## Mídia e dados

Download HTTPS com hosts autorizados, controle de redirects e bloqueio de destinos privados, loopback, link-local e metadata cloud após resolução DNS. Restringir MIME/tamanho, gerar nome seguro e validar bytes. Verificar todos os saltos de redirect; não registrar URLs assinadas. Downloads não podem acessar a rede administrativa.

Arquivos enviados à Gupshup devem ficar acessíveis pelo prazo necessário por URL temporária autorizada, sem publicar permanentemente documentos do atendimento. Definir TTL após medir tempo de busca do provedor; não reutilizar credenciais do Chatwoot na URL. Apagar cópias temporárias conforme retenção.

Segredos em variáveis protegidas/secrets de container, tokens separados por ambiente e usuário técnico com acesso somente necessário à conta/inbox. Endpoints administrativos exigem credencial separada, auditoria e limites. TLS em todas as chamadas externas. Logs com IDs técnicos e telefone mascarado; conteúdo/payload bruto fora dos logs comuns.
