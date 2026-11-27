---
paths:
  - "**/*.repository.ts"
  - "**/*.repository.int.ts"
  - "**/repositories/**"
---

# React: Repositories

A Repository is a plain class that delegates to Data Sources and returns Contracts.

## Layout

```text
src/repositories/
  api/
    orders/
      orders.repository.int.ts
      api-orders.repository.ts
  local/
    orders/
      local-orders.repository.int.ts
      local-orders.repository.ts
```

| Kind | Implementation | Interface |
|---|---|---|
| Remote | `api/<entity>/api-<entity>.repository.ts`, class `Api<Entity>Repository` | `<entity>.repository.int.ts`, type `<Entity>Repository` |
| Local | `local/<entity>/local-<entity>.repository.ts`, class `Local<Entity>Repository` | `local-<entity>.repository.int.ts`, type `Local<Entity>Repository` |

Use Cases depend on the interface, so tests can pass a fake without touching a Data Source.

## Remote shape

```ts
// orders.repository.int.ts
import type { AddOrderRequestDto } from "@/contracts/remote/api/orders/add-order-request.dto";
import type { OrderDto } from "@/contracts/remote/api/orders/order.dto";

export interface OrdersRepository {
  getOrder(id: string): Promise<OrderDto>;
  listOrders(): Promise<OrderDto[]>;
  addOrder(request: AddOrderRequestDto): Promise<OrderDto>;
}
```

```ts
// api-orders.repository.ts
import { Endpoints } from "@/config/endpoints";
import { OrderDtoSchema, type OrderDto } from "@/contracts/remote/api/orders/order.dto";
import type { HttpClient } from "@/data-sources/http/http-client.data-source.int";
import type { OrdersRepository } from "./orders.repository.int";

export class ApiOrdersRepository implements OrdersRepository {
  constructor(private readonly http: HttpClient) {}

  async getOrder(id: string): Promise<OrderDto> {
    const json = await this.http.get(Endpoints.order(id));
    return OrderDtoSchema.parse(json);
  }

  async listOrders(): Promise<OrderDto[]> {
    const json = await this.http.get(Endpoints.orders);
    return OrderDtoSchema.array().parse(json);
  }

  async addOrder(request: AddOrderRequestDto): Promise<OrderDto> {
    const json = await this.http.post(Endpoints.orders, request);
    return OrderDtoSchema.parse(json);
  }
}
```

## Local shape

```ts
// local-orders.repository.ts
import { toOrderDto } from "@/contracts/mappers/to-dtos/order-record.mapper";
import type { OrderDto } from "@/contracts/remote/api/orders/order.dto";
import type { LocalDatabase } from "@/data-sources/local-database/local-database.data-source.int";
import type { LocalOrdersRepository as LocalOrdersRepositoryInt } from "./local-orders.repository.int";

export class LocalOrdersRepository implements LocalOrdersRepositoryInt {
  constructor(private readonly db: LocalDatabase) {}

  async getByUserId(userId: string): Promise<OrderDto[]> {
    const records = await this.db.query("orders", { userId });
    return records.map(toOrderDto);
  }
}
```

## Rules

- Take Data Sources through the constructor, typed to their interfaces. Never import `axios`, `firebase` or any other SDK.
- Parse JSON with the Contract's zod schema, and map records here. Nothing above this layer sees raw payloads.
- Make no business decisions. Preconditions and "which data" choices belong to Use Cases.
- Keep remote and local Repositories separate, one per entity.
- Hold no cache unless it must survive between readers. View caching belongs to TanStack Query in the ViewModel layer.
- Import no React, Zustand or TanStack Query code.
