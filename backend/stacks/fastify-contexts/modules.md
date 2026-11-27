---
paths:
  - "**/*.module.ts"
  - "**/plugins/**"
  - "**/bootstrap.ts"
  - "**/env.schema.ts"
  - "**/hooks/**"
---

# Fastify Contexts: Modules, Plugins and Boot

## Module shape

A Module is the composition root of one context. It builds the object graph by hand and registers the controllers.

```ts
// src/contexts/orders/orders.module.ts
import type { FastifyPluginAsync } from 'fastify';

import { OrderService } from './application/order.service.js';
import { OrderRepository } from './infra/order.repository.js';
import { internalOrderController } from './presentation/http/internal-order.controller.js';
import { orderController } from './presentation/http/order.controller.js';

export const OrdersModule: FastifyPluginAsync = async (app) => {
  const service = new OrderService({
    orders: new OrderRepository(app.uow),
    app,
  });

  await app.register(orderController(service));
  await app.register(internalOrderController(service), { prefix: '/internal/orders' });
};
```

- The Module is a plain `FastifyPluginAsync`, not wrapped in `fastify-plugin`, so its routes stay encapsulated under its prefix.
- Register it in `src/bootstrap.ts` with its prefix: `await app.register(OrdersModule, { prefix: '/orders' })`. A Module that serves two prefixes (a public one and `/internal/...`) is registered without a prefix and sets each one itself.
- A Repository of another context may be constructed here and passed to a Service as its interface.
- When a test or a plugin needs the same graph, export a `create<Name>Service(app)` factory from the Module file and call it in both places.
- No business logic and no route declarations in the Module.

## Boot order

`src/bootstrap.ts` registers everything in a fixed order, documented at the top of `bootstrapApplication()`:

```text
env -> zod compilers -> cookies, cors -> openapi -> database -> redis -> auth -> session
-> rate limit -> events -> Integration plugins -> registerHooks
-> Modules -> events wiring -> schedulers and watchers -> API reference UI
```

- A new Infrastructure plugin goes before the Modules, after the plugins it depends on.
- A new Module goes in the Modules block.
- Something that needs Modules already built (event wiring, a scheduler) goes after them.

## Infrastructure plugins

```ts
// src/plugins/email.plugin.ts
import fp from 'fastify-plugin';

import { loadEnv } from '../env.schema.js';
import { ConsoleEmailSender, type EmailSender, ResendEmailSender } from '../lib/email/index.js';

declare module 'fastify' {
  interface FastifyInstance {
    email: EmailSender;
  }
}

export const EmailPlugin = fp(
  async (app) => {
    const env = loadEnv();
    const email: EmailSender = env.RESEND_API_KEY
      ? new ResendEmailSender(app.log, { apiKey: env.RESEND_API_KEY, from: env.EMAIL_FROM })
      : new ConsoleEmailSender(app.log);

    app.decorate('email', email);
  },
  { name: 'email-plugin' },
);
```

- Wrap with `fastify-plugin` (`fp`), so the decoration is visible to every Module.
- Augment `FastifyInstance` in the same file, typed with the Integration's interface, never the vendor class.
- Give the plugin a `name`, and list the plugins it needs in `dependencies`, such as `['redis-plugin']`.
- Pick the implementation from the environment. Without credentials, fall back to a console or no-op implementation of the same interface, so local runs work.
- Close what you open in `app.addHook('onClose', ...)`: connections, queues, cron tasks.
- A shared Service used by several contexts (such as an entitlement check) may be built in a plugin and decorated on `app`, like any other app-wide instance.

## Environment

- `src/env.schema.ts` holds `EnvSchema` and `loadEnv()`, which parses `process.env` once and caches it. Nothing else reads `process.env`.
- Plugins call `loadEnv()`. Modules read `app.env`. Services receive the values they need through their `deps`.
- A new variable goes into `EnvSchema`, with a default or `.optional()` when it is not required everywhere, and into `.env.example`.
- Use `APP_ENV` to tell environments apart. `NODE_ENV` stays `production`.

## System entries

| Entry | Where | Shape |
|---|---|---|
| Domain-event handler | `application/handlers/`, wired in `src/plugins/events-wiring.plugin.ts` | `app.events.on('OrderPlaced', createOnOrderPlaced({ invoices }), { critical? })` |
| Queue producer | A plugin that decorates `app.<name>Queue` | BullMQ `Queue` with default attempts, backoff, and retention |
| Queue worker | The Module that owns the Service doing the work | BullMQ `Worker` with concurrency from `app.env`, closed in `onClose` |
| Cron | A plugin, enabled by an env flag | `node-cron` task that skips overlapping ticks and takes a Redis lock when only one instance may run it |

- `events-wiring.plugin.ts` registers after the Modules and declares `dependencies: ['events-plugin', 'database-plugin']`. It is the only place that subscribes handlers.
- Job ids are deterministic (`<kind>:<id>:<day>`), so a retry or a second instance never duplicates the work.
- A cron or worker calls Services for the work. Do not add Repository logic to plugins; see departure 4 in `stack/stack-profile.md`.

## Hooks

`src/hooks/index.ts` sets the error handler (`handleHttpException`) and the not-found handler once. Feature code never calls `setErrorHandler`.
