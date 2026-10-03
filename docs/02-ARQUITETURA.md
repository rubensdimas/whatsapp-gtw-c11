# Arquitetura

## Desenho aprovado e escolhas propostas

O usuário confirmou gateway independente do provedor, Gupshup inicial e Chatwoot como interface humana, sem IA agora. Para executar esse desenho, propõe-se monólito modular Python com API FastAPI, worker separado e PostgreSQL próprio. As escolhas de bibliotecas e versões serão fixadas na implementação.

```mermaid
flowchart TD
  W[WhatsApp] <--> G[Gupshup]
  G -->|Webhooks| A[Gateway HTTP]
  C[Chatwoot 4.10.1] -->|Webhooks| A
  A -->|Commit antes do ACK| D[(PostgreSQL do gateway)]
  D <--> P[Worker e núcleo de conversas]
  P -->|Application API| C
  P -->|Adapter Gupshup| G
  P -->|Cópia de mídia E3| M[(Volume de mídia)]
  G -->|GET /media/token| A
  A --> M
  H[Atendentes] <--> C
  T[TI do Conselho] -->|CLI e /admin| A
```

Chatwoot e gateway rodam no mesmo cluster Swarm. O webhook do Chatwoot usa a rede overlay interna; somente as rotas Gupshup e /media são publicadas no Traefik.

## Componentes

| Componente | Responsabilidade |
| --- | --- |
| Webhook controllers | Resolver canal pelo slug, validar segredo da rota, validar envelope, gravar evento e devolver ACK |
| Normalizador | Converter payload externo em mensagem/status internos; separar IDs e unidades de tempo |
| ConversationService | Localizar contato/vínculo, serializar criação e selecionar conversa |
| MessageService | Planejar partes de envio, janela, correlação e efeitos no Chatwoot |
| TextFormatter | Converter markdown e dividir texto longo; funções puras |
| CommandService | Interpretar `/template` e `/templates` em nota privada e o endpoint REST equivalente |
| MediaStore | Guardar cópia temporária de mídia de saída e servir URL assinada (E3) |
| CLI | Canais, segredos de rota, tokens administrativos e catálogo de templates |
| GupshupProvider | send_text, send_media, send_template, parse_message e parse_status |
| ChatwootClient | Operações de contatos, vínculos, conversas, mensagens e status |
| Repositories | Transações, unicidade, eventos, tarefas e mapeamentos |
| Worker | Claim de tarefas, leases, execução, retentativa segura e reconciliação |

## Fila inicial

Usar tarefas PostgreSQL com claim transacional, bloqueio de linha e SKIP LOCKED. Cada claim tem lease; o worker confirma a conclusão em nova transação. Criar webhook_event e job no mesmo commit. Não manter transação de banco aberta durante HTTP externo.

O dispatcher considera ordenação por channel_id + identidade WhatsApp. Não iniciar a segunda mensagem da mesma direção enquanto a anterior estiver executando ou em resultado ambíguo. Resultado ambíguo retém a fila de saída do contato por no máximo ORDERING_HOLD_SECONDS; depois a mensagem recebe failed com aviso e a fila segue ([ADR 0003](adr/0003-retencao-da-fila-por-contato.md)). Canais/contatos diferentes podem avançar em paralelo. Redis poderá ser introduzido mediante necessidade medida; PostgreSQL permanece fonte de verdade.

## Entrada

Validar app/canal → persistir evento → ACK → deduplicar → normalizar → atualizar última mensagem válida do usuário → localizar contato/vínculo → selecionar conversa → criar incoming → registrar ID Chatwoot → concluir tarefa.

O source_id da mensagem incoming terá uma chave externa estável, distinta do source_id do vínculo contact_inbox. Ela ajuda a reconciliar uma criação cujo retorno se perdeu; não é uma garantia de unicidade remota.

## Saída

Persistir evento Chatwoot → validar conta/inbox → nota privada de agente iniciada por `/template(s)` segue para CommandService; demais notas são ignoradas → validar outgoing/private → recuperar conversa e contato (conversa desconhecida segue o [ADR 0004](adr/0004-conversa-iniciada-pelo-atendente.md)) → converter formatação e planejar partes → validar janela e conteúdo → criar tentativa durável → chamar Gupshup → gravar messageId → aguardar recibos. Eventos message_updated não acionam novos envios.

## Status

Guardar recibo bruto → correlacionar gsId e WhatsApp ID → guardar evento não correlacionado se necessário → aplicar progressão local → sincronizar mensagem Chatwoot. Falha na sincronização de status agenda somente atualização do Chatwoot, jamais reenvio da mensagem ao WhatsApp.

## Conversas e identidade

O contato representa uma pessoa; o vínculo identifica sua participação em uma inbox. O adapter é propriedade do canal, não da identidade global da pessoa. Usar source_id estável do vínculo como wa:<channel_uuid>:<wa_id> e identifier do contato como wa:<wa_id>. O nono dígito nunca é inserido/removido no envio nem gravado no Chatwoot; a variante serve apenas para busca ([ADR 0002](adr/0002-resolucao-de-contato.md)). Preservar wa_id recebido e separar dele o telefone de exibição.

Ordem de resolução de contato: channel_contact mapeado → identifier → phone_number exato → variante do nono dígito → criação. No Chatwoot 4.10.1 phone_number e identifier são únicos por conta; a ambiguidade só ocorre entre candidatos de passos diferentes. Nesse caso, usar o passo mais forte, entregar a mensagem e criar aviso privado com os candidatos para merge.

Para contato conhecido, validar o estado remoto da conversa antes de reutilizar. open, pending e snoozed são reutilizáveis; nova incoming deve resultar em atendimento aberto, conforme comportamento homologado. resolved inicia uma nova conversa. Se houver múltiplas não resolvidas, escolher a mais recente e registrar inconsistência para revisão, sem resolver outras automaticamente.

## Limites

Chatwoot é fonte de verdade para atendentes, equipes e estado da conversa. Gateway é fonte de verdade para transporte, jobs e IDs externos. Gupshup/WhatsApp fornecem recibos de transporte. A integração não depende do banco interno do Chatwoot. Nenhum componente de IA será criado agora.
