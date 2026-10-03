# ADR 0005 — Segredo na rota dos webhooks

Data: 03/10/2026 · Status: Aceita

## Contexto

A segurança dos callbacks previa segredo na URL, mas as rotas documentadas não tinham segredo. A Gupshup não oferece, nas referências consultadas, assinatura nem header customizado no callback. O Chatwoot roda no mesmo cluster Swarm do gateway.

## Decisão

Rotas `POST /webhooks/gupshup/{channel_slug}/{secret}` e `POST /webhooks/chatwoot/{channel_slug}/{secret}`. O segredo (≥ 32 bytes aleatórios) é gerado por CLI, guardado apenas como hash no channel, comparado em tempo constante e rotacionável com dois segredos válidos durante a troca. Traefik e aplicação mascaram o segmento nos access logs. O Chatwoot chama o gateway pela rede overlay interna do Swarm; a rota Chatwoot não é publicada no Traefik. O gateway relê a mensagem pela Application API antes de enviar.

## Consequências

Segredo errado responde 404 sem criar evento. A allowlist de IPs Gupshup continua desejável e depende de lista oficial do suporte. Se a conta oferecer mecanismo melhor, novo ADR o substitui.
