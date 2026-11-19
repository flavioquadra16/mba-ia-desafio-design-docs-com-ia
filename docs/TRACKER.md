# Tracker de rastreabilidade

Cada linha liga um item dos design docs à fala da reunião ou a um arquivo que já existe no OMS. Onde o FDD precisou de uma constante que a reunião não numerou (tamanho do lote, comprimento da secret, codificação do HMAC, nome do header da secret anterior), a linha diz isso no resumo e a localização continua sendo a decisão que a constante implementa.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B pediram aviso de status: Atlas Comercial, MaxDistribuição e Nova Cargo | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Problema | Hoje eles consultam a listagem de pedidos para descobrir mudança | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Risco | A Atlas pode migrar se a feature não sair até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-DEF-01 | docs/PRD.md | Requisito Não Funcional | Abaixo de 10 segundos já é tempo real para esses clientes | TRANSCRICAO | [09:02] Marcos |
| PRD-DEF-02 | docs/PRD.md | Restrição | O fluxo é só de saída; o cliente não envia webhook para a plataforma | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Métrica | Meta: primeira tentativa de entrega em menos de 10 segundos no caminho feliz | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Métrica | Meta: 100% das transições com inscrito gravam o evento na mesma transação | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-03 | docs/PRD.md | Métrica | Prazo de três sprints, com fim de novembro para a Atlas | TRANSCRICAO | [09:46] Larissa |
| PRD-RF-01 | docs/PRD.md | Requisito Funcional | POST de cadastro; a plataforma gera a secret e a devolve na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-RF-02 | docs/PRD.md | Requisito Funcional | customerId vai no request; o JWT não identifica o customer | TRANSCRICAO | [09:32] Larissa |
| PRD-RF-03 | docs/PRD.md | Requisito Funcional | Listar, editar e remover os webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-RF-04 | docs/PRD.md | Requisito Funcional | Filtro por status na inserção da outbox; sem inscrito, não insere | TRANSCRICAO | [09:34] Bruno |
| PRD-RF-05 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas, com resultado, payload, resposta e duração | TRANSCRICAO | [09:34] Marcos |
| PRD-RF-06 | docs/PRD.md | Requisito Funcional | Rotação de secret com a anterior válida por 24 horas | TRANSCRICAO | [09:21] Sofia |
| PRD-RF-07 | docs/PRD.md | Requisito Funcional | HMAC-SHA256 do corpo, secret exclusiva por endpoint | TRANSCRICAO | [09:22] Sofia |
| PRD-RF-08 | docs/PRD.md | Requisito Funcional | URL http é recusada na validação | TRANSCRICAO | [09:23] Sofia |
| PRD-RF-09 | docs/PRD.md | Requisito Funcional | Cinco retentativas com esperas 1m, 5m, 30m, 2h e 12h, depois guarda de falha | TRANSCRICAO | [09:17] Diego |
| PRD-RF-10 | docs/PRD.md | Requisito Funcional | Replay manual só com role ADMIN, e a plataforma loga quem executou | TRANSCRICAO | [09:36] Sofia |
| PRD-RF-11 | docs/PRD.md | Requisito Funcional | Garantia at-least-once com identificador estável no X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| PRD-RF-12 | docs/PRD.md | Requisito Funcional | Payload com a transição e total_cents, sem os itens do pedido | TRANSCRICAO | [09:43] Diego |
| PRD-RF-13 | docs/PRD.md | Requisito Funcional | Endpoint tem estado ativo; inativo não recebe evento | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Polling de 2 segundos; no pior caso a agenda espera esse piso | TRANSCRICAO | [09:10] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Teto de 64KB; acima disso é erro, sem truncar | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 segundos na chamada ao destino conta como falha | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-05 | docs/PRD.md | Restrição | Ordem só por pedido, e só enquanto houver um worker | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-06 | docs/PRD.md | Restrição | HTTP do cliente não entra na transação de mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-OUT-01 | docs/PRD.md | Fora de escopo | E-mail de alerta quando o webhook falha fica para fase futura | TRANSCRICAO | [09:37] Larissa |
| PRD-OUT-02 | docs/PRD.md | Fora de escopo | Dashboard visual fica com o time de frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-OUT-03 | docs/PRD.md | Fora de escopo | Rate limit de saída será observado e decidido depois | TRANSCRICAO | [09:39] Larissa |
| PRD-OUT-04 | docs/PRD.md | Fora de escopo | Arquivar linhas entregues depois de 30 dias não entra nesta feature | TRANSCRICAO | [09:08] Diego |
| PRD-OUT-05 | docs/PRD.md | Fora de escopo | Webhook de entrada não faz parte do escopo | TRANSCRICAO | [09:02] Marcos |
| PRD-OUT-06 | docs/PRD.md | Fora de escopo | Exactly-once foi recusado | TRANSCRICAO | [09:25] Diego |
| PRD-OUT-07 | docs/PRD.md | Fora de escopo | Itens do pedido não vão no aviso | TRANSCRICAO | [09:43] Diego |
| PRD-OUT-08 | docs/PRD.md | Fora de escopo | A criação do pedido não dispara webhook; o gancho é changeStatus | TRANSCRICAO | [09:40] Bruno |
| PRD-OUT-09 | docs/PRD.md | Fora de escopo | Vários workers em paralelo ficam para o futuro | TRANSCRICAO | [09:13] Diego |
| PRD-RISK-01 | docs/PRD.md | Risco | Cliente com manutenção de cerca de duas horas; retry longo o bastante para cobrir | TRANSCRICAO | [09:16] Diego |
| PRD-RISK-02 | docs/PRD.md | Risco | Secret já vazou em log de cliente; secret por endpoint e rotação | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Ameaça de saída da Atlas se o prazo do trimestre estourar | TRANSCRICAO | [09:00] Marcos |
| PRD-DEP-01 | docs/PRD.md | Dependência | Sofia revisa HMAC e geração de secret por pelo menos dois dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Marcos documenta a deduplicação no portal de desenvolvedores | TRANSCRICAO | [09:26] Marcos |
| PRD-ACC-01 | docs/PRD.md | Critério de aceitação | A secret aparece na criação e não nas consultas seguintes | TRANSCRICAO | [09:31] Marcos |
| PRD-ACC-02 | docs/PRD.md | Critério de aceitação | Cadastro com http é recusado | TRANSCRICAO | [09:23] Sofia |
| PRD-ACC-03 | docs/PRD.md | Critério de aceitação | Quem não é ADMIN não reprocessa a DLQ | TRANSCRICAO | [09:36] Sofia |
| PRD-ACC-04 | docs/PRD.md | Critério de aceitação | Payload acima do teto não é truncado nem enviado | TRANSCRICAO | [09:23] Sofia |
| RFC-ALT-01 | docs/RFC.md | Alternativa | HTTP síncrono dentro do changeStatus descartado: trava status e não pode dar rollback | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa | Redis Streams ou Cluster descartado: infra extra para um time pequeno | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa | Trigger MySQL descartada: não notifica processo externo | TRANSCRICAO | [09:09] Diego |
| RFC-OPEN-01 | docs/RFC.md | Questão em aberto | Rate limit de saída fica em observação, sem desenho nesta fase | TRANSCRICAO | [09:39] Larissa |
| RFC-OPEN-02 | docs/RFC.md | Questão em aberto | CRUD segue para qualquer role autenticada; endurecer fica para depois | TRANSCRICAO | [09:37] Sofia |
| RFC-OPEN-03 | docs/RFC.md | Questão em aberto | Partição por order_id ou lock pessimista não foi escolhida | TRANSCRICAO | [09:13] Diego |
| RFC-PROP-01 | docs/RFC.md | Decisão | Fechamento: outbox MySQL, worker 2s, retry, DLQ, HMAC, at-least-once, padrões do projeto | TRANSCRICAO | [09:48] Larissa |
| RFC-PROP-02 | docs/RFC.md | Restrição | Estimativa de três sprints incluindo a revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL na mesma transação da mudança de status | TRANSCRICAO | [09:08] Larissa |
| ADR-001-UUID | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Id da outbox é UUID, como o restante do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-001-FN | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | publishWebhookEvent recebe o tx; não injeta o repository inteiro | TRANSCRICAO | [09:41] Bruno |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Decisão | Backoff 1m, 5m, 30m, 2h, 12h, quase 15 horas entre a primeira falha e a última tentativa | TRANSCRICAO | [09:17] Diego |
| ADR-002-DLQ | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Decisão | Tabela webhook_dead_letter com payload, motivo e timestamp, mais replay admin | TRANSCRICAO | [09:18] Diego |
| ADR-002-ALT3 | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Alternativa | Três tentativas descartadas: curtas demais para manutenção de duas horas | TRANSCRICAO | [09:16] Diego |
| ADR-002-ALTINF | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Alternativa | Retry indefinido descartado: evento fica pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256, secret por endpoint, sem secret global | TRANSCRICAO | [09:21] Sofia |
| ADR-003-ROT | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | Rotação com grace period de 24 horas | TRANSCRICAO | [09:21] Sofia |
| ADR-003-HDR | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | Header X-Signature com o HMAC do corpo | TRANSCRICAO | [09:20] Sofia |
| ADR-004 | docs/adrs/ADR-004-at-least-once-com-x-event-id.md | Decisão | At-least-once; o cliente deduplica pelo X-Event-Id gerado na outbox | TRANSCRICAO | [09:25] Diego |
| ADR-004-ALT | docs/adrs/ADR-004-at-least-once-com-x-event-id.md | Alternativa | Exactly-once descartado: exigiria coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| ADR-005 | docs/adrs/ADR-005-worker-polling-em-processo-separado.md | Decisão | Worker em processo separado, polling de 2 segundos, entrypoint novo | TRANSCRICAO | [09:11] Larissa |
| ADR-005-ORD | docs/adrs/ADR-005-worker-polling-em-processo-separado.md | Limitação | Ordenação implícita por order_id só no single-worker; sem ordem global | TRANSCRICAO | [09:12] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Módulo no padrão atual, AppError, Pino, Zod e prefixo WEBHOOK_ | TRANSCRICAO | [09:30] Larissa |
| ADR-006-ADMIN | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Replay reutiliza requireRole de ADMIN | TRANSCRICAO | [09:36] Larissa |
| ADR-006-PRISMA | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | PrismaClient novo por processo, mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisão | Payload renderizado na inserção, não na hora do envio | TRANSCRICAO | [09:52] Larissa |
| ADR-007-ALT | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Alternativa | Guardar só order_id e renderizar no envio foi a opção que Bruno colocou e a call descartou | TRANSCRICAO | [09:51] Bruno |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Inserir na outbox dentro da transação de changeStatus; falha desfaz o status | TRANSCRICAO | [09:41] Diego |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Worker lê pendentes mais antigos a cada 2 segundos | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Tabela operacional do backoff até a DLQ na falha da 5ª retentativa | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | Replay recoloca o evento na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | Um único worker preserva a ordem por created_at daquele pedido | TRANSCRICAO | [09:12] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST de webhook com url, lista de status e customer no corpo | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | PATCH para editar o cadastro | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | GET do histórico de entregas do webhook | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST outbound com event_id, event_type, status, customer_id e total_cents | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | Endpoint para o cliente pedir secret nova | TRANSCRICAO | [09:21] Sofia |
| FDD-ERRO-01 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL quando o esquema não é https | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-02 | docs/FDD.md | Erro | Família de códigos com prefixo WEBHOOK_, exemplo WEBHOOK_NOT_FOUND | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE acima de 64KB, sem truncar | TRANSCRICAO | [09:24] Diego |
| FDD-ERRO-04 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT aos 10 segundos | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-05 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED foi exemplo de nome e não virou caso do contrato | TRANSCRICAO | [09:28] Bruno |
| FDD-HDR-01 | docs/FDD.md | Contrato | Header X-Webhook-Id com o id do cadastro | TRANSCRICAO | [09:45] Sofia |
| FDD-HDR-02 | docs/FDD.md | Contrato | Header X-Timestamp com o instante do envio | TRANSCRICAO | [09:44] Diego |
| FDD-HDR-03 | docs/FDD.md | Contrato | Headers X-Event-Id, X-Signature e Content-Type application/json | TRANSCRICAO | [09:44] Diego |
| FDD-STATUS-01 | docs/FDD.md | Decisão | Estados da outbox: pendente, processando, falhou, entregue | TRANSCRICAO | [09:08] Diego |
| FDD-IDX-01 | docs/FDD.md | Decisão | Índice em status e created_at; worker lê lote pequeno | TRANSCRICAO | [09:08] Diego |
| FDD-WORKER-ENTRY | docs/FDD.md | Decisão | Entrypoint src/worker.ts e script npm run worker, arquivo ainda a criar | TRANSCRICAO | [09:11] Larissa |
| FDD-FILTRO | docs/FDD.md | Decisão | O filtro de status ocorre na inserção, não na hora de mandar | TRANSCRICAO | [09:34] Diego |
| FDD-DETALHE | docs/FDD.md | Contrato | Cliente que quiser itens consulta o GET do pedido | TRANSCRICAO | [09:43] Diego |
| FDD-CRUD-ROLE | docs/FDD.md | Restrição | CRUD de configuração aceita qualquer role autenticada nesta fase | TRANSCRICAO | [09:37] Sofia |
| FDD-RES-01 | docs/FDD.md | Resiliência | Linha presa em processando volta depois do timeout de 10s; a reunião não desenhou reaper | TRANSCRICAO | [09:42] Diego |
| FDD-BATCH | docs/FDD.md | Restrição | Lote pequeno; a constante 10 é default deste FDD, não número da reunião | TRANSCRICAO | [09:08] Diego |
| FDD-SIG-PREV | docs/FDD.md | Contrato | X-Signature-Previous implementa o grace de 24h; o nome do header não foi dito | TRANSCRICAO | [09:21] Sofia |
| FDD-SECRET-LEN | docs/FDD.md | Restrição | Secret gerada pela plataforma; 32 bytes em hex é default do FDD para a revisão da Sofia | TRANSCRICAO | [09:31] Marcos |
| FDD-SAME-ID | docs/FDD.md | Decisão | Replay recoloca o mesmo evento, com o event_id nascido na outbox | TRANSCRICAO | [09:18] Diego |
| FDD-NO-EMAIL | docs/FDD.md | Fora de escopo | Não há canal de fallback por e-mail | TRANSCRICAO | [09:37] Larissa |
| FDD-MODULO | docs/FDD.md | Decisão | Pasta src/modules/webhooks no mesmo formato dos outros domínios | TRANSCRICAO | [09:27] Bruno |
| FDD-LOGGER | docs/FDD.md | Decisão | Log continua no Pino; sem biblioteca nova | TRANSCRICAO | [09:29] Bruno |
| FDD-MW | docs/FDD.md | Decisão | O middleware de erro atual absorve AppError, Zod e Prisma | TRANSCRICAO | [09:29] Bruno |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus é a transação que passará a chamar publishWebhookEvent | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | A máquina de status existente não é reimplementada pelo módulo de webhook | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Erros novos seguem AppError e as subclasses com errorCode estável | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Envelope de erro { error: { code, message, details } } já está no middleware | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | authenticate e requireRole já distinguem ADMIN e OPERATOR | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | User não tem customerId; o customer não pode sair do JWT | CODIGO | prisma/schema.prisma |
| FDD-INT-07 | docs/FDD.md | Integração | Rotas da API já nascem em /api/v1; o limite de JSON de entrada é 1mb | CODIGO | src/app.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Logger Pino com redact de authorization, password e token | CODIGO | src/shared/logger/index.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Listagem paginada reutiliza paginated(); deliveries não, porque o teto é 100 | CODIGO | src/shared/http/response.ts |
| FDD-INT-10 | docs/FDD.md | Integração | createPrismaClient abre uma instância por processo | CODIGO | src/config/database.ts |
| FDD-INT-11 | docs/FDD.md | Integração | validate middleware converte ZodError em VALIDATION_ERROR | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-12 | docs/FDD.md | Integração | createOrderSchema já recebe customerId no corpo e pagina com teto 100 | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-INT-13 | docs/FDD.md | Integração | requireRole ADMIN já compõe a rota de usuário; o replay segue o mesmo arranjo | CODIGO | src/modules/users/user.routes.ts |
| FDD-INT-14 | docs/FDD.md | Integração | src/server.ts é o entrypoint cujo formato o worker separa, sem listen | CODIGO | src/server.ts |
| FDD-INT-15 | docs/FDD.md | Integração | X-Request-Id e requestId já correlacionam log da API | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-16 | docs/FDD.md | Integração | OrderStatus do schema é a lista fechada de eventTypes | CODIGO | prisma/schema.prisma |
| FDD-INT-17 | docs/FDD.md | Integração | Rotas de pedidos são agregadas em buildApiRouter, padrão do router novo | CODIGO | src/routes/index.ts |
| FDD-TEST-01 | docs/PRD.md | Teste | Suíte atual é Vitest e Supertest contra /api/v1/orders | CODIGO | tests/orders.test.ts |
| FDD-TEST-02 | docs/FDD.md | Teste | Script test roda Vitest; engines pedem Node >= 20, onde o fetch do worker existe | CODIGO | package.json |
