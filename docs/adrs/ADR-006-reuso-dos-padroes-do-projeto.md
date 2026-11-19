# ADR-006: Reuso dos padrões já existentes no OMS

## Status

Aceito

- **Data:** 2026-11-18
- **Decisores:** Bruno, Larissa, Diego, Sofia
- **Fonte:** `TRANSCRICAO.md` ([09:27]–[09:37], [09:40]–[09:41]) e o código citado abaixo

## Contexto

O OMS já tem uma forma estável de organizar domínio, erro, validação, autenticação e log. Bruno pediu que webhooks não abram um estilo paralelo. A revisão do módulo fica mais curta se controller, service, repository, routes e schemas Zod continuarem no mesmo lugar, e se o middleware de erro que já entende `AppError`, `ZodError` e Prisma continuar valendo sem ramo novo.

Pontos do código que a feature encontra:

- Módulos em `src/modules/<domínio>/` com controller, service, repository, routes e schemas. Pedidos estão em `src/modules/orders/`.
- `AppError` (`src/shared/errors/app-error.ts`) e erros nomeados em `src/shared/errors/http-errors.ts`, com `errorCode` estável (`INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`).
- `errorMiddleware` (`src/middlewares/error.middleware.ts`) serializa esses erros, Zod e os códigos Prisma `P2002` e `P2025`.
- `authenticate` e `requireRole` em `src/middlewares/auth.middleware.ts`. A role `ADMIN` já protege `GET /users/:id` em `src/modules/users/user.routes.ts`. O JWT carrega `ADMIN` ou `OPERATOR`.
- Logger Pino em `src/shared/logger/index.ts`, com redact de senha e token.
- Validação Zod via `src/middlewares/validate.middleware.ts`.
- `PrismaClient` criado em `src/config/database.ts`, uma instância por processo.
- Rotas montadas em `src/app.ts` sob `/api/v1` e agregadas em `src/routes/index.ts`.
- O model `User` em `prisma/schema.prisma` não tem `customerId`. O `customerId` de um pedido viaja no corpo ou no filtro, como em `src/modules/orders/order.schemas.ts`.

## Decisão

O módulo novo é `src/modules/webhooks/`, com a mesma fatia dos outros domínios: controller, service, repository, routes e schemas. O worker só entra como entrypoint `src/worker.ts` mais o processador dentro do módulo.

Erros do módulo usam o prefixo `WEBHOOK_` em classes que estendem `AppError` (ou as subclasses já existentes, quando o caso é o mesmo de outro domínio). Códigos citados como família: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e equivalentes. `UNAUTHORIZED` e `FORBIDDEN` continuam os códigos gerais do middleware de autenticação.

O endpoint de replay usa `requireRole('ADMIN')` do jeito que `user.routes.ts` já usa. O CRUD de configuração fica aberto a qualquer role autenticada nesta fase. Endurecer isso ficou para depois.

Log segue no Pino já configurado. Não entra biblioteca nova de log. O worker cria o próprio `PrismaClient` com `createPrismaClient()`, mesma `DATABASE_URL`, porque é outro processo. Não compartilha objeto em memória com a API.

A integração com pedidos é a função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada de dentro da transação de `OrderService.changeStatus` (`src/modules/orders/order.service.ts`). Sem injetar o repository de webhooks no service de pedidos.

## Alternativas Consideradas

### Módulo com estrutura própria e cliente Prisma único compartilhado entre API e worker

Descartada. Bruno e Diego concordaram com o layout atual de módulo e com instância de `PrismaClient` por processo. Um singleton em memória não atravessa dois processos Node, então “reutilizar o mesmo objeto” não é possível no desenho de entrypoints separados (ADR-005).

### Injetar `WebhookRepository` inteiro em `OrderService`

Descartada na própria reunião. Bruno propôs a função que recebe o `tx`; Diego reforçou que não é necessário injetar o repository. A transação continua dona da atomicidade.

## Consequências

### Positivas

- Quem já lê um módulo de pedidos consegue navegar o módulo de webhooks sem mapa novo.
- Erros `WEBHOOK_*` saem no envelope `{ error: { code, message, details } }` sem mudança no `errorMiddleware`.
- A autorização de replay reaproveita a única checagem de role que o projeto já tem, em vez de um filtro ad hoc.
- Como `User` não referencia `Customer`, o desenho não tenta “descobrir” o customer pelo JWT. O `customerId` vai explícito no contrato, alinhado ao que Larissa corrigiu na reunião e ao que `createOrderSchema` já faz.

### Negativas

- O módulo herda os limites do middleware atual. `validate` transforma todo `ZodError` em `VALIDATION_ERROR`. Um código `WEBHOOK_INVALID_URL` precisa nascer de um `AppError` do módulo, não de uma esperança de que o middleware compartilhado mude só para webhooks.
- Qualquer usuário autenticado altera webhook de qualquer customer até a questão de autorização ser fechada. Não há, no schema, um vínculo usuário–customer para restringir isso sem decisão nova.
- O redact do Pino hoje não cita secret de webhook. A implementação precisa estender a lista; senão o reuso do logger vaza justamente o que o ADR-003 manda proteger.
