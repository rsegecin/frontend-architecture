# Stack Profile: Fastify + Bounded Contexts

Node 22 ESM with strict TypeScript: Fastify 5 with `fastify-type-provider-zod`, zod 3 Contracts, Kysely 0.27 over `pg` for every query, Prisma 7 only for the schema and migrations (`prisma-kysely` generates the Kysely types), a workspace `rich-domain` package for entities and events, Redis with BullMQ for cache and queues, `node-cron` for schedules, Vitest for tests, tsup for the build, Biome for lint and format.

Everything here is required for every API on this stack, small or large. A small API simply has fewer contexts, and creates `domain/` events, ports, and workers only when it needs them.

## Role map

Paths in the last column are relative to `.claude/rules/backend-architecture/`.

| Core role | Concrete form | Location | Read first |
|---|---|---|---|
| Route | Controller factory `<name>Controller(service)` returning a `FastifyPluginAsync`, routes on `app.withTypeProvider<ZodTypeProvider>()` | `src/contexts/<context>/presentation/http/<name>.controller.ts` | `stack/controllers.md` |
| Contract | zod schema and type with the same name: `<Name>Input`, `<Name>Output` | `presentation/http/dto/input/*.input.ts`, `presentation/http/dto/output/*.output.ts` | `stack/controllers.md` |
| Application Service | Class `<Name>Service` with constructor injection. Event handlers are `create<Handler>(deps)` factories | `src/contexts/<context>/application/`, `application/handlers/` | `stack/services.md` |
| Domain | `Entity` or `Aggregate` subclasses with `create()` and `restore()`, `ValueObject`, `DomainEvent`, pure functions | `src/contexts/<context>/domain/` | `stack/domain.md` |
| Repository | Interface `I<Name>Repository` and class `<Name>Repository` over the unit of work, with to-domain and to-persistence adapters | `src/contexts/<context>/infra/<name>.repository.ts`, `infra/adapters/` | `stack/repositories.md` |
| Integration | Interface named for the capability, vendor implementations, decorated on `app` by a plugin | `src/lib/<capability>/`, or `application/ports/` with `infra/adapters/` inside one context | `stack/modules.md` |
| System entry | Event handlers wired in `events-wiring.plugin.ts`, BullMQ workers started in their Module, cron plugins | `application/handlers/`, `<context>.module.ts`, `src/plugins/` | `stack/modules.md` |
| Module | `<Context>Module: FastifyPluginAsync`, registered in `bootstrap.ts` with its prefix | `src/contexts/<context>/<context>.module.ts` | `stack/modules.md` |
| Infrastructure | `fastify-plugin` plugins that decorate `app`: `env`, `db`, `uow`, `redis`, `cache`, `events`, `email`, Integrations, queue producers | `src/plugins/*.plugin.ts`, `src/lib/database/database.plugin.ts` | `stack/modules.md` |
| Util | Pure functions | `src/lib/util/`, `src/lib/<capability>/` | `core/logic-placement.md` |
| Shared Contracts | Exception classes and the error filter; `ErrorCode`, shared enums, ids, and pagination schemas | `src/lib/errors/`, the workspace `shared-schemas` package | `stack/controllers.md` |

Tests follow `stack/tests.md`. Before adding a dependency, Read `stack/packages-by-layer.md`.

## How this stack applies the core chain

```text
HTTP path:    <name>.controller.ts -> <Name>Service -> I<Name>Repository -> Kysely -> Postgres
                                                   -> app.<integration>  -> external API
System path:  worker / event handler / cron plugin -> <Name>Service or handler -> Repositories
```

- Every context has the same four folders: `presentation/`, `application/`, `domain/`, `infra/`. Create a folder only when its first file is added.
- A Module is the only place that calls `new` on a Repository or Service of its context.
- Transactions open in the Service with `app.uow.withTransaction`. Repositories pick up the active transaction on their own.
- A domain event lets another context react without the caller depending on it. See `stack/services.md`.

## Errors and pagination

- Error body: `OutputError`, with `url`, `method`, `timestamp`, `category`, `code`, `tracerId`, `message`, `statusCode`, `metadata`.
- Validation status: 422, code `VALIDATION`.
- Database conflicts: the error filter turns SQL state `23505` into 409 `ALREADY_EXISTS`, and `55P03` or `23503` into 409 `CONFLICT`. A Service that wants a specific code catches the unique violation itself.
- Pagination: `PaginationQuery` (`page`, `pageSize`) for the query and `paginated(item)` (`data`, `page`, `pageSize`, `total`) for the response, both from the workspace `shared-schemas` package.

## Where this stack departs from core

These are deliberate. Do not refactor them toward `core/` unless the user asks.

1. Repository interfaces live next to their implementation in `infra/`, not in `domain/`. Services import them with `import type`.
2. Services reach Infrastructure through `app` decorations. Older Services take the whole `FastifyInstance`; new ones declare a narrow `<Name>App` type with only the decorations they use. See `stack/services.md`.
3. Exceptions are separate classes that extend `Error`, matched by `instanceof` in `src/lib/errors/http-exception.filter.ts`. A new exception class needs a branch there.
4. The `scheduler` and `retention` cron plugins read Repositories directly to select and enqueue work. Do not add another plugin like them without asking; a new job calls an Application Service.
5. Domain rules throw `DomainError`. `rich-domain` also exports an `Either` (`ok`, `fail`), which is rarely used; do not introduce it where a throw works.

## Naming

Files are kebab-case. Classes and types are PascalCase; variables, functions, and controller factories are camelCase. Imports are relative and end in `.js`.

| Role | File | Export |
|---|---|---|
| Module | `src/contexts/orders/orders.module.ts` | `OrdersModule` |
| Controller | `presentation/http/order.controller.ts` | `orderController` |
| Input Contract | `presentation/http/dto/input/create-order.input.ts` | `CreateOrderInput` (schema and type) |
| Output Contract | `presentation/http/dto/output/order.output.ts` | `OrderOutput`, `OrderListOutput` |
| Service | `application/order.service.ts` | `OrderService` |
| Event handler | `application/handlers/on-order-placed.ts` | `createOnOrderPlaced` |
| Port | `application/ports/payment-gateway.ts` | `interface PaymentGateway` |
| Entity | `domain/entities/order.ts` | `Order` |
| Value object | `domain/value-objects/email.ts` | `Email` |
| Domain event | `domain/events/order-placed.ts` | `OrderPlaced`, event name `"OrderPlaced"` |
| Repository | `infra/order.repository.ts` | `IOrderRepository`, `OrderRepository` |
| Adapters | `infra/adapters/order-to-domain.adapter.ts`, `order-to-persistence.adapter.ts` | `OrderToDomainAdapter`, `OrderToPersistenceAdapter` |
| Plugin | `src/plugins/email.plugin.ts` | `EmailPlugin`, decorates `app.email` |

Tables and columns are snake_case in the database. Adapters map them to camelCase Domain properties.

## Directory map

```text
src/
  main.ts                    # loads env, boots, listens; --generate-openapi-and-exit
  bootstrap.ts               # plugin and Module registration order
  env.schema.ts              # EnvSchema and loadEnv(): the only process.env reader
  hooks/                     # error handler and not-found handler
  plugins/                   # Infrastructure, Integrations, schedulers, event wiring
  contexts/<context>/
    <context>.module.ts
    presentation/http/       # controllers, dto/input, dto/output
    application/             # services, handlers/, ports/
    domain/                  # entities/, value-objects/, events/, pure rules
    infra/                   # repositories, adapters/, vendor adapters, email templates
  lib/
    database/                # Kysely, unit of work, Prisma schema and migrations
    errors/                  # exceptions, OutputError, http-exception.filter.ts
    guards/                  # Guardian
    <capability>/            # email, cache, redis, ai, util...
```

## Commands

| Task | Command |
|---|---|
| Generate OpenAPI | `pnpm generate:openapi` |
| Database migration | `pnpm db:migrate` |
| Generate Kysely types | `pnpm db:generate` |
| Analyze | `pnpm typecheck` in the API, `pnpm lint` at the repository root |
| Test | `pnpm test` |
