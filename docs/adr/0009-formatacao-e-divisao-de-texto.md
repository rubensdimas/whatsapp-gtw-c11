# ADR 0009 — Conversão de markdown e divisão de texto

Data: 03/10/2026 · Status: Aceita

## Contexto

Atendentes escrevem markdown no Chatwoot, que o WhatsApp não interpreta igualmente. Textos podem exceder o limite do WhatsApp.

## Decisão

Converter markdown básico conforme a tabela em [03-CONTRATOS-API](../03-CONTRATOS-API.md#formatação-do-texto-de-saída) (`**x**`→`*x*`, itálico→`_x_`, `~~x~~`→`~x~`, código para crase tripla, link para `texto (url)`) e remover marcação sem equivalente, preservando o texto. Textos acima de TEXT_MAX_CHARS (default 4096, confirmar na referência vigente) são divididos em partes ordenadas usando message_part, preferindo quebras de parágrafo, depois de frase, depois de palavra.

## Consequências

Status agregado segue as regras de múltiplas partes. A conversão é função pura com testes de tabela.
