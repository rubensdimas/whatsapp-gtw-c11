# Escopo e requisitos do MVP

## Resultado esperado

Um profissional envia uma mensagem ao número WhatsApp de teste, ela aparece na Inbox API do Chatwoot e a resposta do atendente chega ao profissional pela Gupshup. Status e erros são rastreáveis. Reenvio do mesmo webhook não duplica mensagens em condições normais; resultados ambíguos são tratados explicitamente.

## Entregas incrementais

| Entrega | Conteúdo | Limite |
| --- | --- | --- |
| E1 — integração básica | Texto nos dois sentidos, contato, vínculo inbox, conversa, persistência, deduplicação e bloqueio fora da janela | Piloto controlado; sem anexos e sem iniciar contato por template |
| E2 — status e operação | Correlação Gupshup/WhatsApp/Chatwoot, recibos, falhas, recuperação, métricas e testes de indisponibilidade | Sem alegação de entrega quando houver apenas aceitação da API |
| E3 — MVP operacional | Imagem, áudio, vídeo, documento e template textual aprovado com parâmetros | Liberação depende da homologação de todos os fluxos |

E1 e E2 são marcos intermediários. O MVP operacional inclui E3. Stickers, reações, localização, contatos, botões complexos e catálogo ficam fora deste MVP; uma mensagem não suportada deve produzir registro e aviso privado ao atendente, sem ser descartada silenciosamente.

## Requisitos funcionais

| ID | Requisito |
| --- | --- |
| RF01 | Receber webhook V2 da aplicação Gupshup configurada e separar message de message-event |
| RF02 | Normalizar texto, identificador WhatsApp e timestamps, preservando o ID externo original |
| RF03 | Reutilizar contato existente após correspondência exata; criar se ausente; associar à Inbox API |
| RF04 | Reutilizar conversa não resolvida da mesma conta/inbox/contato; criar nova após resolução |
| RF05 | Inserir texto/mídia recebidos como incoming público no Chatwoot |
| RF06 | Receber message_created e encaminhar somente outgoing público válido |
| RF07 | Armazenar IDs de envio e atualizar status/falhas no Chatwoot |
| RF08 | Controlar janela de atendimento a partir da última mensagem do usuário, revalidando antes do envio |
| RF09 | Oferecer envio explícito de template aprovado por endpoint administrativo autenticado |
| RF10 | Preservar texto e anexos de uma mensagem como partes rastreáveis e manter sua ordem |
| RF11 | Informar falhas definitivas ou bloqueios ao atendente por status failed e aviso privado deduplicado |
| RF12 | Permitir inspeção e reprocessamento técnico com autorização, auditoria e prevenção de reenvio incerto |

## Requisitos não funcionais

O ACK deve ocorrer após commit durável, com meta local p95 ≤ 1 segundo sob carga de homologação. O fluxo deve sobreviver à reinicialização da API e do worker. Testar inicialmente rajadas de 10 eventos/segundo durante 60 segundos, como carga proposta, não como volume observado do Conselho.

A meta de processamento de texto, excluídas indisponibilidades externas, é p95 ≤ 5 segundos no piloto. Falhas devem ter correlation_id, categoria, tentativas e ação operacional. Segredos e dados pessoais não podem constar em logs comuns.

## Fora do escopo

Agente de IA, integração com sistemas de negócio, autenticação PF/PJ, cobrança, RAG, campanhas em massa, migração automática de histórico, alterações no Chatwoot, vários provedores implementados e alta disponibilidade entre regiões.

## Aceite

E3 só será liberada quando os cenários obrigatórios de [homologação](08-HOMOLOGACAO.md) passarem em Chatwoot 4.10.1 e app Gupshup autorizado. Antes disso, as entregas são restritas ao piloto e às capacidades validadas.
