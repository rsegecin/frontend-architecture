---
paths:
  - "**/contracts/**"
---

# React: Contracts

`src/contracts/` holds the Contracts role. Files here import no other layer and no SDK. The only package they may import is `zod`.

## Layout

```text
src/contracts/
  errors/              # typed errors that cross layers
  signals/             # dependency-free request and change signals
  local/
    domain/            # app-level models
    enums/
    records/           # local persistence records
  remote/
    api/<feature>/     # request and response DTOs, by feature
    models/            # DTOs shared across features
  mappers/
    to-domain/
    to-dtos/
    to-records/
```

## DTOs

Files end in `.dto.ts`. Export the schema as `<Name>DtoSchema` and the type as `<Name>Dto`, inferred from the schema.

```ts
// src/contracts/remote/api/orders/order.dto.ts
import { z } from "zod";

export const OrderDtoSchema = z.object({
  id: z.string(),
  total: z.number(),
  createdAt: z.coerce.date(),
});

export type OrderDto = z.infer<typeof OrderDtoSchema>;
```

Repositories call `OrderDtoSchema.parse(json)` at the boundary. Never declare a DTO type by hand next to its schema.

## Enums

Use `as const` objects with a derived union, so they work with `z.enum`:

```ts
export const OrderStatus = { open: "open", paid: "paid" } as const;
export type OrderStatus = (typeof OrderStatus)[keyof typeof OrderStatus];
export const OrderStatusSchema = z.enum(["open", "paid"]);
```

## Mappers

A mapper is a module of pure functions, in the `mappers/` folder named after its target form.

```ts
// src/contracts/mappers/to-domain/order.mapper.ts
import type { OrderDto } from "../../remote/api/orders/order.dto";
import type { Order } from "../../local/domain/order";

export function toOrder(dto: OrderDto): Order {
  return { id: dto.id, total: dto.total, createdAt: dto.createdAt };
}
```

## Errors

Errors that cross layers extend one base class in `src/contracts/errors/`. Never import an HTTP SDK here; the Data Source converts SDK failures into these types.

```ts
export abstract class AppError extends Error {
  abstract readonly kind: string;
}

export class NetworkError extends AppError {
  readonly kind = "network";
}

export class ApiError extends AppError {
  readonly kind = "api";
  constructor(readonly status: number, message: string) {
    super(message);
  }
}
```

## Signals

A signal is a dependency-free publish/subscribe object. Lower layers raise it; a coordinator or a ViewModel-layer bridge subscribes.

```ts
// src/contracts/signals/data-change.signal.ts
export type DataChange = { domain: "orders" | "expenses"; reason: string };

export class DataChangeSignal {
  private listeners = new Set<(change: DataChange) => void>();

  publish(change: DataChange): void {
    for (const listener of this.listeners) listener(change);
  }

  subscribe(listener: (change: DataChange) => void): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }
}
```
