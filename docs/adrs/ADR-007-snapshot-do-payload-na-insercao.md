# ADR-007: Snapshot do payload na inserção da outbox

## Status

Aceito

- **Data:** 2026-11-18
- **Decisores:** Larissa, Diego, Bruno
- **Fonte:** `TRANSCRICAO.md` ([09:43], [09:51]–[09:52])

## Contexto

O worker envia o HTTP depois do commit, às vezes minutos ou horas depois, por causa do polling e do backoff. Nesse intervalo o pedido pode sofrer outra transição. Faltava decidir se a linha da outbox guarda o JSON pronto ou só a chave do pedido, montando o corpo na hora do envio.

Diego já tinha fechado o formato do corpo: `event_id`, `event_type` `order.status_changed`, timestamp ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido, exemplificados por `total_cents`. Itens do pedido ficam de fora. Quem precisar do detalhe consulta o pedido.

## Decisão

O payload é renderizado na inserção, dentro da transação de `changeStatus`, e gravado na outbox. O worker envia esse snapshot. Ele não relê o pedido para descobrir status ou total na hora do POST.

O timestamp do payload é o instante da inserção. O header `X-Timestamp` continua sendo o instante do envio, porque os dois relógios respondem perguntas diferentes: quando o fato ocorreu e quando esta tentativa saiu.

## Alternativas Consideradas

### Guardar só `order_id` e renderizar no envio

Descartada. Larissa apontou o caso esquisito: se o pedido muda de novo antes do POST, o evento deixa de descrever a transição que o gerou. Diego e Bruno fecharam o snapshot na inserção.

### Incluir os itens do pedido no snapshot

Descartada na definição do formato, antes mesmo desta decisão de momento. Diego tirou os itens para o corpo não inflar, e deixou o `GET` do pedido como caminho para o detalhe. O teto de 64KB reforça o mesmo cuidado, mas não é o motivo original do corte.

## Consequências

### Positivas

- Cada evento permanece fiel à transição que o produziu, mesmo que o retry saia no dia seguinte, com o pedido já em outro status.
- O worker não depende de join com `orders` para montar o contrato externo. A linha da outbox basta para assinar e enviar.
- O corpo permanece pequeno, dentro do exemplo de campos que a reunião listou.

### Negativas

- O JSON pode divergir do estado atual do pedido. O cliente que precisa do presente chama o `GET` do pedido. O webhook não é uma projeção viva.
- A linha fica maior do que uma chave estrangeira. O teto de 64KB é avaliado em cima desse JSON, na inserção. Estourar o teto é erro, sem truncar.
- Um replay reenvia o snapshot original, não um recálculo. Isso é o comportamento desejado da decisão, e precisa estar claro para quem operar a DLQ.
