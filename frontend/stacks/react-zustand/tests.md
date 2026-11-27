---
paths:
  - "**/*.test.ts"
  - "**/*.test.tsx"
  - "**/test/helpers/**"
---

# React: Tests

Vitest with the `jsdom` environment, and React Testing Library for hooks and components.

- `test/` mirrors `src/`. A test for `src/use-cases/orders/add-order.use-case.ts` goes in `test/use-cases/orders/add-order.use-case.test.ts`.
- Shared fakes live in `test/helpers/`, named `fake-*.ts`.
- Fake at the Data Source or Repository boundary, and run the real Use Case against it.

## Fakes

Implement the interface, so no SDK runs.

```ts
// test/helpers/fake-orders-repository.ts
export class FakeOrdersRepository implements OrdersRepository {
  orders: OrderDto[] = [];
  error: Error | null = null;

  async getOrder(id: string): Promise<OrderDto> {
    if (this.error) throw this.error;
    const order = this.orders.find((o) => o.id === id);
    if (!order) throw new ApiError(404, "not found");
    return order;
  }

  async listOrders(): Promise<OrderDto[]> {
    if (this.error) throw this.error;
    return this.orders;
  }

  async addOrder(request: AddOrderRequestDto): Promise<OrderDto> {
    const order = { id: String(this.orders.length + 1), total: 0, createdAt: new Date() };
    this.orders.push(order);
    return order;
  }
}
```

## Use Case tests

Construct the class directly. No React, no Zustand.

```ts
it("publishes a change after adding an order", async () => {
  const repository = new FakeOrdersRepository();
  const changes = new DataChangeSignal();
  const published = vi.fn();
  changes.subscribe(published);

  await new AddOrderUseCase(repository, changes).execute({ items: [{ sku: "a", qty: 1 }] });

  expect(repository.orders).toHaveLength(1);
  expect(published).toHaveBeenCalledWith({ domain: "orders", reason: "order-added" });
});
```

## ViewModel tests

```tsx
// test/helpers/render-view-model.tsx
export function renderViewModel<T>(hook: () => T, overrides: Partial<DataSources> = {}) {
  const container = createContainer(overrides);
  const queryClient = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: React.ReactNode }) => (
    <WiringProvider container={container}>
      <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
    </WiringProvider>
  );
  return { ...renderHook(hook, { wrapper }), container, queryClient };
}
```

```ts
it("maps a server error to a translation key", async () => {
  const http = new FakeHttpClient({ error: new ApiError(500, "boom") });
  const { result } = renderViewModel(useOrderListScreenViewModel, { httpClient: http });

  await waitFor(() => expect(result.current.errorKey).toBe("errors.server"));
});
```

- Override Data Sources through `createContainer`. Repositories and Use Cases are built on them, so the real chain runs against the fake.
- `retry: false` makes a failing query settle in its error state at once.
- Create a fresh container and `QueryClient` in every test, so no instance or cache leaks between tests.
- Component tests render the View with the same wrapper and assert on what the user sees, by role and text.

## Resetting view-state stores

A `*.state.ts` store is a module-level singleton, so its state would leak from one test into the next. Zustand's testing guide solves this with a mock that records every store's initial state and restores it after each test.

```ts
// __mocks__/zustand.ts, in the Vitest root
import { act } from "@testing-library/react";
import { afterEach, vi } from "vitest";
import type * as ZustandExportedTypes from "zustand";
export * from "zustand";

const { create: actualCreate, createStore: actualCreateStore } =
  await vi.importActual<typeof ZustandExportedTypes>("zustand");

export const storeResetFns = new Set<() => void>();

const createUncurried = <T>(stateCreator: ZustandExportedTypes.StateCreator<T>) => {
  const store = actualCreate(stateCreator);
  const initialState = store.getInitialState();
  storeResetFns.add(() => store.setState(initialState, true));
  return store;
};

// Supports both create(fn) and the curried create<T>()(fn).
export const create = (<T>(stateCreator: ZustandExportedTypes.StateCreator<T>) =>
  typeof stateCreator === "function"
    ? createUncurried(stateCreator)
    : createUncurried) as typeof ZustandExportedTypes.create;

const createStoreUncurried = <T>(stateCreator: ZustandExportedTypes.StateCreator<T>) => {
  const store = actualCreateStore(stateCreator);
  const initialState = store.getInitialState();
  storeResetFns.add(() => store.setState(initialState, true));
  return store;
};

export const createStore = (<T>(stateCreator: ZustandExportedTypes.StateCreator<T>) =>
  typeof stateCreator === "function"
    ? createStoreUncurried(stateCreator)
    : createStoreUncurried) as typeof ZustandExportedTypes.createStore;

afterEach(() => {
  act(() => storeResetFns.forEach((resetFn) => resetFn()));
});
```

```ts
// test/setup.ts, listed in `test.setupFiles` in vitest.config.ts
import { vi } from "vitest";

vi.mock("zustand");
```

- The `__mocks__` directory sits in Vitest's `root`. If the config changes `root`, move the directory with it.
- Stores import `create` from `"zustand"`, so the mock sees every one of them. No test resets a store by hand.
- The reset also clears the boot flag in `initial-boot-complete.state.ts`, so each test starts in phase 1.
- A test sets view state the way a user would, through the action a ViewModel returns, wrapped in `act`.
