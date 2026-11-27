---
paths:
  - "**/package.json"
---

# Fastify Contexts: Packages by Layer

This guide lists the packages this stack uses for each layer in `core/layers.md`, as `stack/stack-profile.md` maps them. Read it before adding a dependency. Dev tooling (Biome, tsup, tsx, type definitions) implements no layer and is left out.

## Packages

| Package | Major version | Layer | Why this layer |
|---|---|---|---|
| zod | 3 | Contract, Domain | Input and output schemas, the environment schema, and value-object validation |
| fastify | 5 | Route, Module, Infrastructure | HTTP server, plugin encapsulation, hooks, decorations |
| fastify-type-provider-zod | 4 | Route | Validates and serializes with the zod Contracts, and feeds the OpenAPI transform |
| fastify-plugin | 5 | Infrastructure | Makes a plugin's decorations visible outside its scope |
| @fastify/swagger, @scalar/fastify-api-reference | 9, 1 | Route tooling | OpenAPI document generated from the routes, and the reference UI |
| @fastify/cors, @fastify/cookie, @fastify/rate-limit | various | Infrastructure | Cross-origin, cookies for the session, per-route rate limits |
| kysely, pg | 0, 8 | Repository | Every query. Repositories are the only callers |
| prisma, prisma-kysely | 7, 3 | Database schema | Schema, migrations, seed, and the generated Kysely `DB` type. Not used for queries |
| workspace `rich-domain` | — | Domain | `Entity`, `Aggregate`, `ValueObject`, `DomainEvent`, `BaseAdapter`, `DomainError` |
| workspace `shared-schemas` | — | Contract, Shared Contracts | Enums, ids, error codes, and pagination shared with the front end |
| better-auth | 1 | Infrastructure | Authentication and sessions, behind the auth and session plugins |
| ioredis | 5 | Infrastructure | Cache and locks |
| bullmq | 5 | System entry | Queues and workers for background jobs |
| node-cron | 3 | System entry | Scheduled ticks |
| Vendor SDKs (stripe, ai, @ai-sdk/*) | various | Integration | Only inside the Integration that wraps them |

## Avoid with this stack

| Package | Why |
|---|---|
| Prisma Client for queries | A second query path that skips the unit of work and its transactions |
| typeorm, sequelize, mikro-orm | A second data layer |
| express, express middleware | A second HTTP framework. Use the Fastify plugin equivalent |
| class-validator, class-transformer, joi, yup | A second validation library next to zod |
| inversify, tsyringe, awilix | DI is manual, in the Modules |
| A second queue or scheduler (agenda, bee-queue) | BullMQ and node-cron already own background work |
| winston, a second logger | Fastify's pino logger is the only logger |

To add a package that is not listed here, first name the layer it belongs to. If it implements a layer, add a row to this file. If it fits no layer and is not a Util or dev tool, raise it with the user before installing it.
