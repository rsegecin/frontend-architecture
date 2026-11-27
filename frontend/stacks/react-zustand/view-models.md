---
paths:
  - "**/*.view-model.ts"
  - "**/*.state.ts"
  - "**/*.controller.ts"
  - "**/query/**"
---

# React: ViewModels

A ViewModel is a hook named `use<View>ViewModel`, in a `*.view-model.ts` file next to the View it serves. It only talks to Use Cases. See `stack/stack-profile.md` for the naming table.

## Query keys

Query keys live in `src/query/keys/`, one factory per domain. The first element is always the domain, so a `DataChangeSignal` can invalidate the whole domain.

```ts
// src/query/keys/orders.keys.ts
export const ordersKeys = {
  all: ["orders"] as const,
  list: () => [...ordersKeys.all, "list"] as const,
  detail: (id: string) => [...ordersKeys.all, "detail", id] as const,
};
```

Use Cases never see query keys.

## Query shape

```ts
// src/screens/orders/order-list-screen.view-model.ts
import { useQuery } from "@tanstack/react-query";
import { ordersKeys } from "@/query/keys/orders.keys";
import { toErrorKey } from "@/query/error-keys";
import { useUseCases } from "@/wiring/wiring-provider";

export function useOrderListScreenViewModel() {
  const { listOrders } = useUseCases();

  const query = useQuery({
    queryKey: ordersKeys.list(),
    queryFn: () => listOrders.execute(),
  });

  return {
    orders: query.data ?? [],
    isLoading: query.isPending,
    errorKey: query.error ? toErrorKey(query.error) : null,
  };
}
```

## Mutation shape

```ts
// src/screens/orders/order-form-screen.view-model.ts
export function useOrderFormScreenViewModel() {
  const { addOrder } = useUseCases();
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: (request: AddOrderRequestDto) => addOrder.execute(request),
    onSuccess: (order) => {
      queryClient.setQueryData(ordersKeys.detail(order.id), order);
      return queryClient.invalidateQueries({ queryKey: ordersKeys.list() });
    },
  });

  return {
    submit: mutation.mutate,
    isSaving: mutation.isPending,
    result: toOrderFormResult(mutation),
  };
}
```

`toOrderFormResult` and the result type live in `order-form-screen.state.ts`.

## Mapping errors

`src/query/error-keys.ts` maps Contract errors to translation keys. ViewModels return keys; Views translate them.

```ts
export function toErrorKey(error: unknown): string {
  if (error instanceof NotSignedInError) return "errors.notSignedIn";
  if (error instanceof NetworkError) return "errors.offline";
  if (error instanceof ApiError && error.status >= 500) return "errors.server";
  return "errors.unknown";
}
```

## Stateful Use Cases

For state a Use Case holds, read it with `useSyncExternalStore`, not with a query:

```ts
export function useAccountMenuViewModel() {
  const { session } = useUseCases();
  const current = useSyncExternalStore(session.subscribe, session.getSnapshot, () => null);
  return { userName: current?.userName ?? null, signOut: () => session.signOut() };
}
```

## Client-only view state

State shared by a View and its children, such as a selected tab or filter, is a Zustand store in the colocated `*.state.ts`. The file keeps the store private and exports selector hooks.

```ts
// src/screens/orders/order-list-screen.state.ts
import { create } from "zustand";

type OrderFilterState = {
  status: "all" | OrderStatus;
  setStatus: (status: "all" | OrderStatus) => void;
};

const useOrderFilterStore = create<OrderFilterState>()((set) => ({
  status: "all",
  setStatus: (status) => set({ status }),
}));

export const useOrderFilterStatus = () => useOrderFilterStore((s) => s.status);
export const useSetOrderFilterStatus = () => useOrderFilterStore((s) => s.setStatus);
```

The ViewModel calls these hooks and returns the values as plain view state. State that only one component needs stays in `useState`; see `stack/views.md`.

1. Never export the store. Export one selector hook per value or action.
2. Always read through a selector. A selector that builds a new object or array on each call re-renders forever; select one value per hook, or wrap the selector in `useShallow` from `zustand/react/shallow`.
3. Store actions only set view state. They never call a Use Case or fetch, and server data never goes in a store; it stays in the query cache.
4. Write to a store only from event handlers and effects, never during render. The store is a module-level singleton, and on the server it is shared by every request.
5. Never use the `persist` middleware. Storage is a Data Source, so a persisted setting goes through a Use Case and its Repository.

## Rules

- Every `queryFn` and `mutationFn` calls a Use Case method. Never call `axios`, a Repository, a Data Source or a Service.
- Get Use Cases from `useUseCases()`. Never `new` one, and never import anything else from `src/wiring/`.
- Invalidate or update the affected keys in `onSuccess`. Never use `refetchInterval` or manual `refetch()` loops to keep data fresh.
- `enabled` only expresses view readiness, such as a missing route param. Business preconditions stay in the Use Case.
- Return plain view state and action functions. Never return the raw `query` or `mutation` object, or an `Error`.
- Hold no refs, timers or form instances. Those are local UI state in the View or its controller.
- Report only view-specific failures that no Use Case owns, through the project's error-reporting facade.
