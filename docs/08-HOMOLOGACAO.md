# Plano de homologação

Nenhum cenário abaixo foi executado contra as contas do Conselho nesta preparação. Este documento define evidências necessárias, não resultados de testes já aprovados.

## Matriz obrigatória

| ID | Cenário | Resultado esperado | Etapa |
| --- | --- | --- | --- |
| H01 | Texto de contato novo | Contato/vínculo/conversa e incoming corretos | E1 |
| H02 | Texto de contato conhecido | Sem duplicar contato ou conversa não resolvida | E1 |
| H03 | Mesmo webhook recebido três vezes | Um efeito externo conhecido | E1 |
| H04 | Dois eventos simultâneos do mesmo contato | Uma criação local de vínculo/conversa; ordem preservada | E1 |
| H05 | Resposta humana | Um envio Gupshup e messageId salvo | E1 |
| H06 | Incoming, nota privada, activity, outra inbox e message_updated | Nenhum envio WhatsApp | E1 |
| H07 | Conversa resolved e novo inbound | Nova conversa correta | E1 |
| H08 | Conversa snoozed/pending | Reuso e atendimento aberto conforme regra | E1 |
| H09 | Janela expirada/sem incoming conhecido | Texto bloqueado, failed e aviso privado | E1 |
| H10 | Worker reinicia após commit/antes do processamento | Evento retomado sem perda | E1 |
| H11 | Banco falha antes do commit | ACK de sucesso não é enviado | E1 |
| H12 | Timeout após POST externo | Resultado incerto, sem reenvio cego | E1 |
| H13 | delivered/read fora de ordem | Status não regride | E2 |
| H14 | Status antes de gravar messageId | Recibo pendente reconciliado | E2 |
| H15 | gsId ausente com WhatsApp ID conhecido | Correlação correta | E2 |
| H16 | failed síncrono e assíncrono | IDs, erro e status failed corretos | E2 |
| H17 | PATCH de status falha | Retry do status, zero reenvios WhatsApp | E2 |
| H18 | 429, 401/403 e 5xx | Políticas distintas, limite e alerta corretos | E2 |
| H19 | Chatwoot cria objeto mas resposta se perde | Reconciliação por identidade/source_id antes de repetir | E2 |
| H20 | read sem delivered anterior | read aceito sem aguardar evento ausente | E2 |
| H21 | Imagem/áudio/vídeo/documento inbound e outbound | Bytes, legenda, nome e tipo corretos | E3 |
| H22 | Mídia expirada/acima do limite | Falha explícita e nenhum download infinito | E3 |
| H23 | Texto + múltiplos anexos | Partes ordenadas e status agregado | E3 |
| H24 | URL privada/redirect malicioso/MIME divergente | Bloqueio sem acesso à rede interna | E3 |
| H25 | Template aprovado fora da janela | Envio explícito com parâmetros e histórico, sem loop | E3 |
| H26 | Repetir comando de template | Mesmo resultado; chave com corpo diferente gera conflito | E3 |
| H27 | Template enviado, usuário não responde | Texto livre permanece bloqueado | E3 |
| H28 | Callback com slug/segredo inválido, app divergente ou segredo antigo após rotação concluída | Rejeitado; sem evento nem job; segredo ausente dos logs | Todas |
| H29 | Evento desconhecido válido e tipo não suportado | Registro/quarentena/aviso conforme contrato | Todas |
| H30 | Rajada proposta e recuperação de backlog | ACK/latência medidos, sem perda conhecida | E2 |
| H31 | Backup e restauração do gateway | Mapeamentos/eventos/jobs recuperados | Liberação |
| H32 | Contato existente com outra variante do nono dígito | Contato reutilizado; envio usa wa_id recebido | E1 |
| H33 | identifier e telefone apontam para contatos distintos | Mensagem entregue ao candidato mais forte; um aviso privado com candidatos | E1 |
| H34 | Envio ambíguo seguido de nova resposta do atendente | Fila retida até o prazo; depois failed "envio incerto", aviso e envio da seguinte | E1 |
| H35 | Atendente inicia conversa para contato sem histórico | wa_id derivado do telefone; texto bloqueado com orientação de template | E1 |
| H36 | Markdown e texto acima do limite | Formatação convertida; partes ordenadas e status agregado | E1 |
| H37 | Rotação de segredo com dois válidos | Ambos aceitos durante a troca; só o novo após concluir | E1 |
| H38 | `/templates`, `/template` válido, chave/parâmetro inválido e nota comum | Lista, envio único, nota de erro, nenhum transporte da nota | E3 |
| H39 | Mesma nota de comando reentregue | Um único envio de template | E3 |
| H40 | Endpoint admin com token revogado e com token válido | Revogado rejeitado; válido registra o ator | E2 |
| H41 | Mídia de saída buscada após TTL | 404; arquivo removido pela limpeza | E3 |
| H42 | Resposta citada inbound | Prefixo com trecho quando a original é conhecida | E1 |

## Níveis de teste

Unitários: parsers, unidades de timestamp, identidade, janela, filtros, correlação e progressão. Integração com PostgreSQL real: unicidade, transações, concorrência, lease e recovery. Contratos HTTP: requests e respostas anonimizados, incluindo retorno perdido. Contrato local: Chatwoot 4.10.1 no compose e payloads do sandbox Gupshup; não substitui a homologação. Ponta a ponta: Chatwoot exatamente 4.10.1 na instalação do Conselho e app/número Gupshup autorizado.

Mocks não comprovam compatibilidade de versão, permissões reais, acesso a mídia ou entrega WhatsApp. Confirmar o corpo retornado após PATCH, não apenas seu HTTP 200.

## Evidência por execução

Registrar cenário, data, versão/digest Chatwoot, build gateway, app de homologação, passos, IDs técnicos, resultado e observações. Redigir payloads e capturas para excluir números reais, tokens e conteúdo pessoal. Medir ACK na camada HTTP e processamento até a criação remota, separadamente.

## Liberação

E1 exige H01–H12, H28, H29, H32–H37 e H42. E2 exige também H13–H20, H30 e H40. E3 exige H21–H27, H38, H39, H41, todos os anteriores e H31. Nenhuma falha crítica de perda, duplicação automática incerta, vazamento de nota privada ou SSRF pode permanecer aberta. Lacunas de origem/autenticação impedem produção.
