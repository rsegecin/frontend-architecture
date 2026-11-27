---
paths:
  - "**/contexts/**/infra/**"
  - "**/lib/database/**"
---

# Fastify Contexts: Repositories and Persistence

Queries use Kysely, typed by the `DB` type that `prisma-kysely` generates from the Prisma schema. Prisma itself runs only migrations and the seed.

## Shape

The interface and the class live in the same file. Services depend on the interface.

```ts
// src/contexts/orders/infra/order.repository.ts
import { KyselyRepository } from '../../../lib/database/index.js';
import type { Order } from '../domain/entities/order.js';
import { type OrderRow, OrderToDomainAdapter } from './adapters/order-to-domain.adapter.js';
import { OrderToPersistenceAdapter } from './adapters/order-to-persistence.adapter.js';

export interface IOrderRepository {
  findByIdForAccount(accountId: string, id: string): Promise<Order | null>;
  findByReference(accountId: string, reference: string): Promise<Order | null>;
  insert(order: Order): Promise<void>;
  update(order: Order): Promise<void>;
  deleteForAccount(accountId: string, id: string): Promise<void>;
}

export class OrderRepository
  extends KyselyRepository<'orders', Order, OrderRow>
  implements IOrderRepository
{
  protected readonly tableName = 'orders' as const;
  protected readonly toDomain = new OrderToDomainAdapter();
  protected readonly toPersistence = new OrderToPersistenceAdapter();

  async findByIdForAccount(accountId: string, id: string): Promise<Order | null> {
    const row = await this.db
      .selectFrom('orders')
      .selectAll()
      .where('id', '=', id)
      .where('account_id', '=', accountId)
      .executeTakeFirst();
    return row ? this.toDomain.build(row as OrderRow) : null;
  }

  async findByReference(accountId: string, reference: string): Promise<Order | null> {
    const row = await this.db
      .selectFrom('orders')
      .selectAll()
      .where('account_id', '=', accountId)
      .where('reference', '=', reference)
      .executeTakeFirst();
    return row ? this.toDomain.build(row as OrderRow) : null;
  }

  async deleteForAccount(accountId: string, id: string): Promise<void> {
    await this.db.deleteFrom('orders').where('id', '=', id).where('account_id', '=', accountId).execute();
  }
}
```

## Base classes

| Base | Use when | Gives |
|---|---|---|
| `KyselyRepository<'table', Entity, Row>` | The table maps to one entity | `findById`, `insert`, `update`, `delete`, `count` through the two adapters |
| `KyselyConnection` | Custom queries: joins, composite keys, read rows | `this.db`, `this.uow`, `applyPagination()`, `parseCount()` |

Both take the `UnitOfWork` in the constructor: `new OrderRepository(app.uow)`.

## Adapters

```ts
// infra/adapters/order-to-domain.adapter.ts
export type OrderRow = {
  id: string;
  account_id: string;
  reference: string;
  quantity: number;
  status: 'open' | 'cancelled';
  created_at: Date;
  updated_at: Date;
};

export class OrderToDomainAdapter extends BaseAdapter<OrderRow, Order> {
  build(row: OrderRow): Order {
    return Order.restore({
      id: row.id,
      accountId: row.account_id,
      reference: row.reference,
      quantity: row.quantity,
      status: row.status,
      createdAt: row.created_at,
      updatedAt: row.updated_at,
    });
  }
}
```

- One adapter per direction: `<Name>ToDomainAdapter` calls `Entity.restore()`, and `<Name>ToPersistenceAdapter` returns the snake_case row.
- Storage quirks are handled here: `pg` returns `Decimal` and `bigint` columns as strings, so convert them to numbers in the to-domain adapter.

## Transactions and locks

- `this.db` returns the active transaction when the Service opened one with `app.uow.withTransaction`, and the root connection otherwise. A Repository never opens its own transaction.
- A lock method (`SELECT ... FOR UPDATE`) first checks `this.uow.getActive()` and throws when there is no transaction, because a lock outside one is released at once.

## Read rows

A join-heavy query that the Service only reads may return a typed row instead of an entity. Export its type from the Repository file, such as `OrderWithCustomerRow`, and alias joined columns with `as`. This avoids N+1 queries in the Service.

## Rules

- Every query for a caller's data filters by the caller's id in SQL.
- No business decisions: "which status counts as active" arrives from the Service, or lives in a clearly named query method.
- No `try/catch` for database errors. The error filter turns unique, lock, and foreign-key violations into 409. A Service that needs a specific code checks before the write, or catches the SQL state itself.
- A table outside this context may be joined for reads. Writes to another context's table go through that context's Repository.

## Schema changes

- Edit the Prisma schema in `src/lib/database/prisma/`, then run `pnpm db:migrate` to create the migration and `pnpm db:generate` to refresh the Kysely `DB` types.
- New tables and columns are snake_case, like the existing ones.
- Migrations follow `core/contracts-and-errors.md`: they must apply to a database with data.
