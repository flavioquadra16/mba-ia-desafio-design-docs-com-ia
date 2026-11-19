# ADR-001: Outbox de webhooks no MySQL

## Status

Aceito

- **Data:** 2026-11-18
- **Decisores:** Larissa, Diego, Bruno
- **Fonte:** reunião registrada em `TRANSCRICAO.md` ([09:03]–[09:08], [09:40]–[09:41], [09:51])

## Contexto

O OMS precisa avisar sistemas externos quando o status de um pedido muda. Hoje `OrderService.changeStatus` já faz, numa única transação Prisma, a atualização de `orders`, a inserção em `order_status_history` e o ajuste de `stock_quantity`. Não existe fila, broker nem tabela de eventos.

A chamada HTTP ao cliente não pode entrar nessa transação. Um endpoint lento seguraria a mudança de status dos demais pedidos, e uma falha do cliente não pode desfazer um status que a operação já decidiu persistir. O time também não quer subir infraestrutura nova: o banco em produção é o MySQL acessado pelo Prisma (`prisma/schema.prisma`).

Na modelagem do registro, o identificador da linha precisa seguir o restante do schema, em que as chaves são UUID.

## Decisão

Adotar o padrão outbox no MySQL já usado pela aplicação.

Quando `changeStatus` confirmar uma transição e existir ao menos um endpoint ativo do `customer` inscrito no status de destino, a mesma transação que grava pedido, histórico e estoque insere uma linha em `webhook_outbox`. O identificador da linha é um UUID e é o `event_id` do evento. Se a transação sofrer rollback, o evento desaparece com ela. Se ela der commit, o evento está durável, mesmo que o processo da API caia em seguida.

A tabela tem os estados pendente, processando, falhou e entregue, com índice em status e em `created_at`. Um worker separado lê só o que está pronto para envio, em lote pequeno, do mais antigo para o mais novo. Arquivar linhas entregues depois de 30 dias fica fora desta feature.

A publicação entra pela função `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe o client da transação atual. `OrderService` não passa a depender do repositório inteiro de webhooks.

## Alternativas Consideradas

### Chamada HTTP síncrona dentro de `changeStatus`

Descartada. A transação atual já é pesada. Um cliente lento bloquearia outras mudanças de status, e não há rollback aceitável se o destinatário estiver fora do ar.

### Redis Streams ou Redis Cluster

Descartada. Resolveria o desacoplamento, com o custo de operar mais um componente. O time é pequeno, e a outbox no MySQL existente cobre a atomicidade sem infra nova. Foi considerado overengineering.

### Identificador autoincremental na outbox

Descartada na confirmação final da modelagem. Larissa fechou UUID para acompanhar o padrão do projeto, em que os ids são `@default(uuid())`.

## Consequências

### Positivas

- O commit do status e o registro do evento acontecem juntos. Não fica status confirmado sem evento, nem evento de uma transação desfeita.
- Nenhuma chamada de rede entra no caminho crítico de `changeStatus`.
- Não há broker novo para provisionar, monitorar ou recuperar.

### Negativas

- A transação de status ganha mais uma escrita. Uma falha ao inserir na outbox desfaz a mudança de status inteira, inclusive estoque e histórico.
- A tabela cresce com o volume de transições. A limpeza das linhas entregues após 30 dias não será feita agora, então a retenção fica em aberto até uma feature posterior.
- A entrega passa a depender de um worker que observa a tabela. A latência mínima inclui o intervalo de polling descrito no ADR-005.
