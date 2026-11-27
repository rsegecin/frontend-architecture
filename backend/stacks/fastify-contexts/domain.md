---
paths:
  - "**/contexts/**/domain/**"
---

# Fastify Contexts: Domain

Domain code builds on the workspace `rich-domain` package: `Entity`, `Aggregate`, `ValueObject`, `DomainEvent`, `Id`, `DomainError`, and `BaseAdapter`.

```text
src/contexts/orders/domain/
  entities/order.ts          # Entity or Aggregate subclasses
  value-objects/email.ts     # ValueObject subclasses
  events/order-placed.ts     # DomainEvent subclasses
  order-rules.ts             # pure functions shared by the context's Services
```

## Entities

```ts
// src/contexts/orders/domain/entities/order.ts
import { DomainError, Entity, type EntityProps, Id } from '@<scope>/rich-domain';

export type OrderProps = EntityProps<{
  accountId: Id;
  reference: string;
  quantity: number;
  status: 'open' | 'cancelled';
}>;

export class Order extends Entity<OrderProps> {
  static create(input: { accountId: string; reference: string; quantity: number }): Order {
    return new Order({
      accountId: new Id(input.accountId),
      reference: input.reference,
      quantity: input.quantity,
      status: 'open',
    });
  }

  static restore(raw: {
    id: string;
    accountId: string;
    reference: string;
    quantity: number;
    status: 'open' | 'cancelled';
    createdAt: Date;
    updatedAt: Date;
  }): Order {
    return new Order({ ...raw, id: new Id(raw.id), accountId: new Id(raw.accountId) });
  }

  get accountId(): Id {
    return this.props.accountId;
  }

  get status(): OrderProps['status'] {
    return this.props.status;
  }

  cancel(): void {
    if (this.props.status === 'cancelled') throw new DomainError('Order is already cancelled');
    this.props.status = 'cancelled';
    this.touch();
  }
}
```

- `create()` builds a new entity; `Entity` generates the `id`, `createdAt`, and `updatedAt`. `restore()` rebuilds one from stored values and is called only by a to-domain adapter.
- State lives in `this.props`. Expose it through getters; change it only through intention-named methods (`cancel()`, `updateDetails()`), each ending with `this.touch()`.
- A broken invariant throws `DomainError`, which the error filter answers with 422.
- An optional static `schema` (zod) on the class is checked on construction. Use it for invariants that are pure shape rules.
- Ids are `Id` value objects inside the Domain. Adapters and Services convert with `.value`.
- `Aggregate` adds an event list (`addEvent`, `dispatchAll`). Most contexts publish events from the Service instead; use either one consistently within a context.

## Value objects

```ts
export class Email extends ValueObject<string> {
  static schema = z.string().email();

  protected override transformBeforeCreate(value: string): string {
    return value.trim().toLowerCase();
  }
}
```

- Immutable, compared by value with `equals()`.
- `transformBeforeCreate` normalizes; `schema` and `validate()` reject invalid values with `DomainError`.

## Domain events

```ts
export type OrderPlacedPayload = { orderId: string; accountId: string };

export class OrderPlaced extends DomainEvent<OrderPlacedPayload> {
  constructor(aggregateId: Id, payload: OrderPlacedPayload) {
    super(aggregateId, payload, 'OrderPlaced');
  }
}
```

- Pass the event name explicitly. Handlers subscribe by that string, and bundling can rename classes.
- The payload holds primitive ids and values, never entities, so a handler in another context does not depend on this context's classes.

## Pure rules

A calculation used by more than one Service of the context, or one worth testing alone, is an exported function in `domain/`, with a test next to it.

## Rules

- Domain imports only `rich-domain`, `zod`, the workspace `shared-schemas` package, and other Domain files of the same context.
- No Kysely, Fastify, environment, Integration, or Contract imports.
- An entity knows nothing about columns or tables. Mapping lives in `infra/adapters/`; see `stack/repositories.md`.
- Use `rich-domain`'s `Either` (`ok`, `fail`) only for rules that must report several failures without throwing.
