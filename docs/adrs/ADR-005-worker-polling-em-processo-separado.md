# ADR-005: Worker em processo separado, com polling de 2 segundos

## Status

Aceito

- **Data:** 2026-11-18
- **Decisores:** Diego, Larissa, Bruno, Marcos
- **Fonte:** `TRANSCRICAO.md` ([09:08]–[09:14])

## Contexto

A outbox (ADR-001) só produz notificação se algum processo ler a tabela e fizer o HTTP. Esse processo precisa enxergar o commit logo o bastante para a meta de produto: abaixo de 10 segundos já é tempo real para os clientes.

O worker não pode viver dentro do processo da API. Um restart da API derrubaria a drenagem da outbox junto com o HTTP de gestão. O projeto já sobe a API por um entrypoint dedicado, `src/server.ts`. O banco e o Prisma são os mesmos; a instância de `PrismaClient` é por processo.

Também ficou a pergunta de ordenação: PAID, depois PROCESSING, depois SHIPPED, em sequência rápida, precisam chegar na ordem daquele pedido. Ninguém pediu ordem global entre pedidos diferentes.

## Decisão

O worker é um processo Node separado, com entrypoint novo `src/worker.ts` e script `npm run worker`. Ele usa a mesma `DATABASE_URL` e uma instância própria de `PrismaClient`. A lógica de processamento fica no módulo de webhooks (`webhook.processor.ts` ou nome equivalente), não no entrypoint.

O loop faz polling a cada 2 segundos, busca os eventos prontos mais antigos e os processa. MySQL não entra como notificador. Com um único worker, a ordem de `created_at` na outbox preserva a ordem por `order_id`. Isso não é garantia de ordenação global. Escalar para vários workers em paralelo sai desta fase. Se um dia for necessário, as opções deixadas na mesa foram particionar por `order_id` ou usar lock pessimista. Nenhuma das duas foi escolhida.

A latência mínima de agendamento, no pior caso, é de 2 segundos. A reunião aceitou esse piso.

## Alternativas Consideradas

### Trigger no MySQL, ou um mecanismo no estilo `NOTIFY`/`LISTEN` do Postgres

Descartada. Diego explicou que o MySQL não tem listener nativo que acorde um processo externo. Uma trigger executa SQL; para avisar o worker seria preciso improvisar arquivo ou chamada HTTP, o que o time considerou esquisito. O polling de 2 segundos cabe na meta de 10 segundos.

### Worker dentro do mesmo processo de `src/server.ts`

Descartada. O restart da API levaria o worker junto e a outbox pararia de ser drenada até o processo voltar. Larissa e Diego fecharam entrypoint separado.

### Vários workers em paralelo nesta fase

Descartada como escopo atual. Diego mostrou que o paralelismo sem partição perde a ordem. Fica como limitação conhecida e como questão em aberto no RFC, não como desenho desta entrega.

## Consequências

### Positivas

- A meta de “abaixo de 10 segundos” no caminho feliz permanece viável: a espera de agenda é de no máximo 2 segundos, mais o tempo de rede.
- Reiniciar a API não apaga o ciclo de entrega.
- O deploy do worker pode seguir o ciclo do processo de API ou não, sem acoplar os dois loops no mesmo event loop.

### Negativas

- Existe um processo a mais para operar, com o mesmo schema e outra conexão no pool.
- A ordem por pedido só vale enquanto houver um worker e enquanto ele não ultrapassar um evento mais antigo ainda não entregue daquele `order_id`. O FDD torna essa regra explícita. Vários workers sem a partição futura quebram a premissa.
- Se o processo morrer com uma linha em “processando”, essa linha não volta sozinha. O FDD define a recuperação mínima amarrada ao timeout de 10 segundos; a reunião não desenhou um reaper separado.
- Throughput fica limitado a um processo. Rate limit de saída também não entra agora (questão em aberto).
