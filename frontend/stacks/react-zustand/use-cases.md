---
paths:
  - "**/*.use-case.ts"
  - "**/use-cases/**"
---

# React: Use Cases

A Use Case is a plain class named `*UseCase`, in `src/use-cases/<domain>/*.use-case.ts`. Group domains as `core/logic-placement.md` describes.

## Shape

```ts
// src/use-cases/orders/export-orders.use-case.ts
import { NotSignedInError } from "@/contracts/errors/app-errors";
import type { CurrentUserRepository } from "@/repositories/local/current-user/current-user.repository.int";
import type { LocalOrdersRepository } from "@/repositories/local/orders/local-orders.repository.int";
import { OrdersExporter } from "@/utils/export/orders-exporter";

export class ExportOrdersUseCase {
  constructor(
    private readonly orders: LocalOrdersRepository,
    private readonly currentUser: CurrentUserRepository,
  ) {}

  async canExport(): Promise<boolean> {
    const user = await this.currentUser.get();
    return Boolean(user?.id);
  }

  async execute(): Promise<void> {
    const user = await this.currentUser.get();
    if (!user?.id) throw new NotSignedInError();

    console.info("[ExportOrdersUseCase] Starting export");
    const orders = await this.orders.getByUserId(user.id);
    await OrdersExporter.share(orders);
  }
}
```

The ViewModel gets the instance from `useUseCases()` and calls the method:

```ts
const { exportOrders } = useUseCases();
useMutation({ mutationFn: () => exportOrders.execute() });
```

## Changing data

A Use Case that changes data publishes a `DataChangeSignal`, so every ViewModel watching that domain refreshes, whether a ViewModel or a Service called it.

```ts
export class AddOrderUseCase {
  constructor(
    private readonly orders: OrdersRepository,
    private readonly changes: DataChangeSignal,
  ) {}

  async execute(request: AddOrderRequestDto): Promise<OrderDto> {
    if (request.items.length === 0) throw new EmptyOrderError();
    const order = await this.orders.addOrder(request);
    this.changes.publish({ domain: "orders", reason: "order-added" });
    return order;
  }
}
```

## Shared state

When consumers must react to state the Use Case holds, such as a session or a device stream, expose `subscribe` and `getSnapshot`. ViewModels read it with `useSyncExternalStore`.

```ts
export class SessionUseCase {
  private session: Session | null = null;
  private listeners = new Set<() => void>();

  getSnapshot = (): Session | null => this.session;

  subscribe = (listener: () => void): (() => void) => {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  };

  private set(next: Session | null): void {
    this.session = next;
    for (const listener of this.listeners) listener();
  }
}
```

`getSnapshot` must return the same reference until the state changes.

## Rules

- Take Repositories, other Use Cases and signals through the constructor, typed to interfaces. Only `src/wiring/` calls `new`.
- Never import a Data Source, a Service, React, Zustand or TanStack Query.
- Own the business rules: preconditions, "the user must be signed in", "which data to fetch". Throw Contract errors when they fail.
- Return Contracts. Keep view-specific strings, formatting and navigation out.
- Call Utils for stateless technical work, such as `OrdersExporter`.
- Keep each Use Case focused on one responsibility. A domain file may export several small Use Case classes.
- Only the boot Use Case may start or stop Services. See `stack/app-boot.md`.
