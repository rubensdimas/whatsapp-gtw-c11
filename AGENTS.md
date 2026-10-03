# Instruções para desenvolvimento

## Contexto e escopo

Leia README.md e os documentos em docs/ antes de implementar. O produto é uma ponte Gupshup ↔ Chatwoot **4.10.1**, com atendimento humano. Não adicionar IA, Agno, RAG, tools de negócio, Implanta.NET ou n8n nesta entrega.

## Regras técnicas

- Separar recepção HTTP, normalização, regras de conversa, persistência e chamadas externas.
- Implementar somente GupshupProvider; uma interface pequena deve permitir outros adapters no futuro sem criar funcionalidades especulativas.
- Usar Application API do Chatwoot e Inbox API. Não misturar Client API ou Platform API sem revisão dos contratos.
- Fixar a compatibilidade em 4.10.1. Conferir código da tag e fixtures reais, pois a documentação online acompanha versões posteriores.
- Persistir o webhook antes de confirmar seu recebimento. Não usar BackgroundTasks como única garantia de entrega.
- Deduplicar por canal e ID externo. Nunca prometer exactly-once entre APIs independentes.
- Usar o source_id efetivamente retornado pelo vínculo contato/inbox e o ID público da conversa na API de contas.
- Considerar somente outgoing público da conta/inbox configuradas como candidato a envio. Ignorar incoming, notas privadas, activity e message_updated para transporte de saída.
- Não supor que submitted/enqueued significa sent; não inferir entrega pelo status padrão do Chatwoot.
- Não reenviar automaticamente POST com resultado ambíguo. Implementar reconciliation_required e intervenção operacional.
- Nunca escrever diretamente no banco do Chatwoot nem modificar seu core para esta integração.

## Segurança e qualidade

Não incluir tokens, telefones reais, CPF, URLs assinadas ou conteúdo de atendimentos em commits/logs. Fixtures devem ser anonimizadas. Tratar mídia como entrada não confiável e prevenir SSRF.

Criar testes úteis para parsing, deduplicação, corridas, falhas de rede, ordem de status e janela de atendimento. Homologar em 4.10.1 antes de declarar compatibilidade. Atualizar a documentação quando o contrato implementado mudar.

## Estrutura sugerida, ainda não criada

app/api, app/providers, app/services, app/repositories, app/models, app/workers, migrations, tests/unit, tests/integration e tests/fixtures. Usar arquivos pequenos por responsabilidade; evitar duplicar regras do domínio em routers e adapters.

## Limite de atuação

Não realizar migração de número, disparos em massa ou alterações em produção a partir destes documentos. Iniciar pelo ambiente de homologação e seguir os critérios do roadmap.
