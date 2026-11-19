# ADRs — Sistema de Webhooks de Notificação de Pedidos

Decisões fechadas na reunião registrada em `TRANSCRICAO.md`. O formato é MADR. O RFC aponta para cá; o FDD implementa o que estes registros fecharam.

| ADR | Decisão |
| --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Outbox no MySQL, na mesma transação de `changeStatus`, id UUID |
| [ADR-002](ADR-002-retry-backoff-e-dlq.md) | Retentativas 1m/5m/30m/2h/12h e DLQ em tabela separada |
| [ADR-003](ADR-003-hmac-sha256-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação de 24h |
| [ADR-004](ADR-004-at-least-once-com-x-event-id.md) | At-least-once com `X-Event-Id` estável |
| [ADR-005](ADR-005-worker-polling-em-processo-separado.md) | Worker em processo separado, polling de 2 segundos |
| [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md) | Módulo, erros, auth, log e Prisma no padrão do OMS |
| [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md) | Payload renderizado na inserção da outbox |
