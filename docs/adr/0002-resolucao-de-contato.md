# ADR 0002 — Resolução de contato, nono dígito e identifier

Data: 03/10/2026 · Status: Aceita

## Contexto

A regra anterior suspendia a vinculação quando a busca fosse ambígua, deixando a mensagem invisível ao atendente. Números brasileiros podem chegar com ou sem o nono dígito, e o Chatwoot pode ter o contato no outro formato. No Chatwoot 4.10.1, phone_number e identifier são únicos por conta (validação e índice em app/models/contact.rb), portanto não é possível criar um segundo contato com o mesmo telefone.

## Decisão

- identifier do contato criado pelo gateway: `wa:<wa_id>`, por pessoa e compartilhado entre canais da conta.
- Ordem de resolução: (1) channel_contact já mapeado; (2) identifier `wa:<wa_id>`; (3) phone_number igual ao wa_id recebido em E.164; (4) variante brasileira com/sem nono dígito; (5) criar contato com phone_number e identifier.
- Quando mais de um candidato aparecer em passos diferentes (identifier aponta para um contato e telefone para outro, ou as duas variantes existem como contatos distintos), usar o candidato do passo mais forte, entregar a mensagem normalmente e criar aviso privado deduplicado listando os IDs candidatos para merge manual no Chatwoot.
- O envio sempre usa o wa_id recebido; a variante serve apenas para busca.
- Não sobrescrever nome, telefone ou identifier de contato existente.

## Consequências

Nenhuma mensagem fica retida por ambiguidade. Duplicatas são tratadas pelo atendente com o merge nativo. A lista de variantes é regra local, testada por unidade, e não normaliza dados no Chatwoot.
