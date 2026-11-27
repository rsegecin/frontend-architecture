---
paths:
  - "**/contexts/**/application/**"
---

# Fastify Contexts: Application Services, Handlers and Ports

## Service shape

A Service is a class. Every dependency arrives in the constructor; new Services take one named `deps` object.

```ts
// src/contexts/orders/application/order.service.ts
import type { FastifyBaseLogger } from 'fastify';

import { ConflictException, ItemNotFound } from '../../../lib/errors/index.js';
import type { UnitOfWork } from '../../../lib/database/index.js';
import type { EmailSender } from '../../../lib/email/index.js';
import { Order } from '../domain/entities/order.js';
import { renderOrderReceivedEmail } from '../infra/email/render-order-received-email.js';
import type { IOrderRepository } from '../infra/order.repository.js';
import type { CreateOrderInput } from '../presentation/http/dto/input/create-order.input.js';
import type { OrderOutput } from '../presentation/http/dto/output/order.output.js';

export type OrderServiceApp = {
  uow: Pick<UnitOfWork, 'withTransaction'>;
  email: Pick<EmailSender, 'send'>;
  log: FastifyBaseLogger;
};

export type OrderServiceDeps = {
  orders: IOrderRepository;
  app: OrderServiceApp;
  now?: () => Date;
};

export class OrderService {
  private readonly now: () => Date;

  constructor(private readonly deps: OrderServiceDeps) {
    this.now = deps.now ?? (() => new Date());
  }

  async create(accountId: string, input: CreateOrderInput): Promise<OrderOutput> {
    const order = await this.deps.app.uow.withTransaction(async () => {
      const existing = await this.deps.orders.findByReference(accountId, input.reference);
      if (existing) throw new ConflictException('Order already exists.', 'ORDER_ALREADY_EXISTS');

      const created = Order.create({ accountId, reference: input.reference, quantity: input.quantity });
      await this.deps.orders.insert(created);
      return created;
    });

    this.notifyInBackground(order);
    return toOrderOutput(order);
  }

  async remove(accountId: string, id: string): Promise<void> {
    const order = await this.deps.orders.findByIdForAccount(accountId, id);
    if (!order) throw new ItemNotFound('Order', id);
    await this.deps.orders.deleteForAccount(accountId, id);
  }

  // Best-effort: a failed email never fails the order.
  private notifyInBackground(order: Order): void {
    void this.deps.app.email
      .send(renderOrderReceivedEmail(order))
      .catch((err) => this.deps.app.log.warn({ err, orderId: order.id.value }, 'order: email failed'));
  }
}
```

## Dependencies

- Repositories arrive as their interface. A Repository from another context arrives narrowed to the methods used: `Pick<IAccountRepository, 'findById'>`.
- App-wide instances (`uow`, `log`, `env` values, `email`, `events`, `cache`, Integrations, shared Services such as `entitlements`) come from `app` decorations. Declare a `<Name>App` type listing only what the Service uses, so tests pass a small fake.
- Older Services receive the whole `FastifyInstance` as `app`. Do not copy that into new Services.
- Pass `now` (and any id or random source) as an optional dependency when a rule depends on time, so tests control it.
- A Service never creates a Repository, Integration, queue, or client, and never calls `loadEnv()`.

## Rules

- Throw the typed exceptions from `src/lib/errors/`. Return nothing to mean "not found".
- Queries for a caller's data pass the caller's id to the Repository (`findByIdForAccount`). A foreign id gives `ItemNotFound`.
- Open a transaction with `app.uow.withTransaction(async () => ...)`. Repository calls inside it share the transaction through `AsyncLocalStorage`, and a nested call reuses the outer transaction.
- A read-check-write that two requests could race locks first: call a Repository lock method (`lockAccountForWrite`, `SELECT ... FOR UPDATE`) inside the transaction.
- External calls and best-effort work run after the transaction, with `void promise.catch(log)`.
- A Service may return output-shaped objects, typed with the context's Output types. Build them in a private `to<Name>Output()` function, not field by field in each method.
- Small pure helpers used by one Service may sit at the top of its file. A helper used by two Services moves to `domain/` (business rule) or `src/lib/util/` (technical).

## Domain events

Publish an event when another context must react, and the caller must not depend on the reaction.

```ts
await this.deps.app.events.publish(new OrderPlaced(order.id, { orderId: order.id.value, accountId }));
```

Handlers are factories in the reacting context's `application/handlers/`:

```ts
// src/contexts/invoices/application/handlers/on-order-placed.ts
export function createOnOrderPlaced(deps: { invoices: IInvoiceRepository }) {
  return async (event: OrderPlaced): Promise<void> => {
    const existing = await deps.invoices.findByOrder(event.payload.orderId);
    if (existing) return;
    await deps.invoices.insert(Invoice.create({ orderId: event.payload.orderId }));
  };
}
```

- Handlers are idempotent: they check before they write, because an event can be published again.
- Wire every handler in `src/plugins/events-wiring.plugin.ts`. See `stack/modules.md`.
- `{ critical: true }` makes the publisher run the handler first, in sequence, and propagate its error to the caller. Use it only when the caller needs the effect. Other handlers run in parallel, and their failures are only logged.
- The publisher is in memory and synchronous with the request. An event is not durable: work that must survive a crash goes through a queue.

## Ports

An external system used by one context gets a port in that context:

- `application/ports/<capability>.ts` holds the interface and the types it speaks, such as `PaymentGateway` with `CheckoutRequest`.
- `infra/adapters/<vendor>-<capability>.ts` implements it, such as `StripePaymentGateway`, plus a `Noop...` implementation for environments without credentials.
- A plugin picks the implementation from the environment and decorates `app`, such as `app.payments`. The Service depends on the interface only.

An external system used by several contexts lives in `src/lib/<capability>/` instead, with the same interface-plus-implementations shape.
