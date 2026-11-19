# RFC: Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| Autor | Larissa (Tech Lead) |
| Status | Em revisão |
| Data | 2026-11-18 |
| Revisores | Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança), Marcos (Produto) |
| Origem | Reunião em `TRANSCRICAO.md`. Larissa combinou abrir o desenho e revisá-lo com Bruno e Diego antes da implementação. |

Este RFC propõe a arquitetura. O detalhe de schema, payload, códigos e sequência do worker está no [FDD](FDD.md). Cada decisão fechada está num ADR em [docs/adrs](adrs/README.md).

## Resumo executivo

Os clientes B2B deixam de descobrir mudança de status por polling e passam a receber um POST outbound quando o pedido transiciona. A escrita do evento ocorre na mesma transação MySQL que persiste o status. Um processo separado faz polling da outbox a cada 2 segundos, assina o corpo com HMAC-SHA256 de uma secret por endpoint e entrega com garantia at-least-once. Falha transitória segue cinco retentativas (1m, 5m, 30m, 2h, 12h). Falha permanente vai para uma DLQ com replay manual de `ADMIN`.

Não entra nesta proposta: e-mail de alerta, painel visual, rate limit de saída, webhook de entrada, exactly-once e mais de um worker.

## Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo consultam a listagem de pedidos para saber se o status mudou. A integração fica lenta e cara. Para eles, abaixo de 10 segundos já conta como tempo real. A Atlas condicionou a permanência na plataforma à entrega até o fim do trimestre; o prazo combinado na reunião é fim de novembro, em três sprints, com ao menos dois dias úteis de revisão de segurança antes do deploy.

O fluxo que muda status hoje é `OrderService.changeStatus`: atualiza `orders`, insere `order_status_history` e mexe no estoque dentro de `prisma.$transaction`. Não há notificação externa. A solução precisa encaixar nessa transação sem colocar rede no meio dela, e sem infra além do MySQL que o OMS já opera.

O tráfego é só de saída. O customer da inscrição não vem do JWT: o model `User` não referencia `Customer`, e a reunião corrigiu o cadastro para receber `customerId` no request, como o pedido já faz.

## Proposta técnica

Módulo `src/modules/webhooks/`, no mesmo formato dos domínios atuais. Quatro blocos:

1. **Configuração.** CRUD autenticado de endpoints (URL https, lista de status, flag ativo, customer explícito). A plataforma gera a secret e só a devolve na criação e na rotação. A secret anterior vale 24 horas depois da rotação. Listagem das últimas 100 entregas de um endpoint.

2. **Outbox.** `publishWebhookEvent(tx, order, fromStatus, toStatus)` roda dentro da transação de `changeStatus`. O filtro de status acontece nessa inserção: sem endpoint ativo interessado naquele status, não há linha. O JSON gravado é o snapshot da transição, não uma releitura posterior do pedido. O id da linha é UUID e viaja como `X-Event-Id`.

3. **Worker.** Processo à parte (`src/worker.ts`, `npm run worker`), polling de 2 segundos, um worker só. Entrega HTTP com timeout de 10 segundos. Ordem garantida por `order_id` enquanto essa premissa de processo único valer. Sem garantia de ordem global.

4. **Falha.** Backoff 1m / 5m / 30m / 2h / 12h, cinco retentativas após o envio inicial, depois tabela `webhook_dead_letter`. Replay: `POST /admin/webhooks/dead-letter/:id/replay`, role `ADMIN`, com log de quem chamou.

A assinatura é HMAC-SHA256 do corpo, secret por endpoint, header `X-Signature`. O contrato público completo, inclusive os demais headers e os códigos `WEBHOOK_*`, fica no FDD para este RFC não virar especificação de implementação.

## Alternativas consideradas

### HTTP síncrono dentro de `changeStatus`

Colocar o POST no meio da transação de status. O trade-off que eliminou a opção: a transação já atualiza pedido, histórico e estoque. Cliente lento travaria outras mudanças, e cliente fora do ar obrigaria um rollback do status, que o time recusou. A outbox separa “o fato foi commitado” de “o HTTP chegou”.

### Redis Streams ou Redis Cluster

Publicar o evento num stream externo em vez da tabela MySQL. O trade-off: ganho de um mecanismo de fila pronto, custo de subir e operar cluster para um time pequeno, com uma atomicidade pior do que a inserção na mesma transação SQL. Outbox no MySQL existente foi considerada suficiente.

### Trigger no MySQL para acordar o worker

Tentar ser mais reativo do que o polling, no estilo `NOTIFY`/`LISTEN`. O trade-off: o MySQL não notifica processo de fora. A trigger só roda SQL; avisar o worker exigiria improviso (arquivo ou HTTP). Polling de 2 segundos já cabe no limite de 10 segundos, então a complexidade extra não compra a meta.

Outras rejeições que não mudam a forma da solução, e por isso não ocupam seção própria: retry de três tentativas (curto demais para a manutenção de duas horas já observada), retry sem teto (evento eterno), secret global (vazamento único atinge todo mundo), exactly-once (coordenação dos dois lados) e renderizar o payload só na hora do envio (o evento deixaria de descrever a transição original). Cada uma está no ADR correspondente.

## Questões em aberto

1. **Rate limit de saída.** Se um customer tiver dezenas de pedidos mudando no mesmo minuto, o worker dispara uma chamada por evento. Diego levantou o ponto; Larissa deixou como observar e decidir depois. Não há limite, fila por destino nem código de “adiar por taxa” nesta proposta.

2. **Autorização do CRUD.** Sofia aceitou, por enquanto, qualquer role autenticada (`ADMIN` ou `OPERATOR`) para cadastrar, editar, listar e remover webhooks. O replay permanece só com `ADMIN`. Não ficou decidido o que “endurecer” significa, e o schema não liga usuário a customer. A proposta não inventa esse vínculo.

3. **Escala além de um worker.** A ordem por pedido depende do processo único. Diego citou partição por `order_id` ou lock pessimista, e adiou a escolha. Esta RFC não escolhe entre as duas. Enquanto a escolha não existir, o desenho operacional é um processo.

Ficam de fora de propósito, sem serem perguntas abertas de arquitetura: e-mail quando o endpoint falha, painel visual e arquivamento das linhas entregues após 30 dias. São fases ou projetos seguintes, não decisões pendentes deste desenho.

## Impacto e riscos

- **Pedidos.** `changeStatus` passa a escrever na outbox na mesma transação. A criação do pedido (`create`, que já nasce `PENDING`) não dispara webhook. Só a transição feita por `changeStatus` dispara.
- **Operação.** Passa a existir um processo worker junto da API, no mesmo banco. Restart da API não drena a outbox; parar o worker, sim.
- **Segurança.** Secrets recuperáveis no banco, assinatura obrigatória, TLS obrigatório na URL de destino. Sofia reserva dois dias úteis de revisão antes do deploy, com atenção a HMAC e à geração da secret.
- **Cliente.** Integração nova, com deduplicação por `X-Event-Id` e verificação de HMAC. Marcos documenta isso no portal. Sem portal, o contrato do FDD ainda é a referência do time.
- **Risco de prazo.** A ameaça de saída da Atlas está amarrada ao fim do trimestre. O escopo acima é o que cabe nas três sprints estimadas. Itens da seção anterior que voltarem para esta fase estouram essa conta.
- **Risco de duplicata e de ordem.** At-least-once produz repetição. Ordem só é prometida por pedido, e só com um worker. Os dois pontos precisam estar no material que o cliente lê.

## Decisões relacionadas

- [ADR-001 — Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Retry, backoff e DLQ](adrs/ADR-002-retry-backoff-e-dlq.md)
- [ADR-003 — HMAC-SHA256 com secret por endpoint](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md)
- [ADR-004 — At-least-once com X-Event-Id](adrs/ADR-004-at-least-once-com-x-event-id.md)
- [ADR-005 — Worker em polling, processo separado](adrs/ADR-005-worker-polling-em-processo-separado.md)
- [ADR-006 — Reuso dos padrões do projeto](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
- [ADR-007 — Snapshot do payload na inserção](adrs/ADR-007-snapshot-do-payload-na-insercao.md)
