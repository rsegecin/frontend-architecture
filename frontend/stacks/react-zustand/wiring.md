---
paths:
  - "**/wiring/**"
  - "**/*.wiring.ts"
---

# React: Dependency Wiring and Reactivity

`src/wiring/` is the only place that creates Data Sources, Repositories, Use Cases, Services and signals. A plain factory, `createContainer`, builds them all once, and React Context hands the result to hooks. No state library is involved, and tests swap the lowest layer through the factory's argument.

## 1. One factory file per layer and domain

```text
src/wiring/
  container.ts                    # createContainer(overrides) and the Container type
  signals.wiring.ts               # createSignals()
  data-sources.wiring.ts          # createDataSources(signals, overrides), DataSources type
  repositories/orders.wiring.ts   # createOrdersRepositories(dataSources)
  use-cases/orders.wiring.ts      # createOrdersUseCases(repositories, signals)
  use-cases/system.wiring.ts      # createSystemInitUseCase(useCases, services)
  services.wiring.ts              # createServices(useCases), createCoordinators(useCases, signals)
  wiring-provider.tsx             # WiringProvider, useUseCases, useCoordinators, useDataChangeSignal
```

A factory file exports functions and types only. It holds no module-level instance.

## 2. The container

```ts
// data-sources.wiring.ts
export type DataSources = { tokenStorage: TokenStorage; httpClient: HttpClient };

export function createDataSources(signals: Signals, overrides: Partial<DataSources> = {}): DataSources {
  const tokenStorage = overrides.tokenStorage ?? new LocalTokenStorage();
  const httpClient =
    overrides.httpClient ?? new AxiosHttpClient(Config.apiBaseUrl, tokenStorage, signals.signOut);
  return { tokenStorage, httpClient };
}
```

An overridden `tokenStorage` is the one the HTTP client receives, so a fake reaches everything built on it.

```ts
// repositories/orders.wiring.ts
export type OrdersRepositories = ReturnType<typeof createOrdersRepositories>;

export function createOrdersRepositories(dataSources: DataSources) {
  const orders: OrdersRepository = new ApiOrdersRepository(dataSources.httpClient);
  return { orders };
}
```

```ts
// use-cases/orders.wiring.ts
export type OrdersUseCases = ReturnType<typeof createOrdersUseCases>;

export function createOrdersUseCases(repositories: OrdersRepositories, signals: Signals) {
  return {
    listOrders: new ListOrdersUseCase(repositories.orders),
    addOrder: new AddOrderUseCase(repositories.orders, signals.dataChange),
  };
}
```

A Use Case's key is its class name in camelCase without the `UseCase` suffix: `AddOrderUseCase` becomes `addOrder`.

```ts
// container.ts
export type UseCases = OrdersUseCases & { systemInit: SystemInitUseCase };

export type Container = {
  useCases: UseCases;
  coordinators: Startable[];
  dataChangeSignal: DataChangeSignal;
};

export function createContainer(overrides: Partial<DataSources> = {}): Container {
  const signals = createSignals();
  const dataSources = createDataSources(signals, overrides);
  const repositories = { ...createOrdersRepositories(dataSources) };
  const domainUseCases = { ...createOrdersUseCases(repositories, signals) };
  const services = createServices(domainUseCases);
  const systemInit = createSystemInitUseCase(domainUseCases, services);
  return {
    useCases: { ...domainUseCases, systemInit },
    coordinators: createCoordinators(domainUseCases, signals),
    dataChangeSignal: signals.dataChange,
  };
}
```

- **Build order** is signals, Data Sources, Repositories, Use Cases, Services, the boot Use Case, then coordinators. A new domain adds one spread to `repositories` and `domainUseCases`, and one type to the `UseCases` intersection.
- **Services and coordinators are lists.** `createServices` and `createCoordinators` each return a `Startable[]`, so adding one changes no caller.
- **Only the return value leaves the factory.** Repositories, Data Sources and non-coordinator Services stay local to `createContainer`, so no ViewModel can reach them.
- **Overrides are for Data Sources only.** Everything above them is always the real class.
- **Construction is eager.** Every instance is built when the container is, including during server pre-rendering. This is safe because constructors do no I/O, touch no browser API and start nothing.
- **`createContainer` is called in two places:** the `useState` initializer in `src/app/providers.tsx`, and test helpers. Never `new` a role class in a component or a hook body.

## 3. The provider and hooks

```tsx
// wiring-provider.tsx
"use client";

const ContainerContext = createContext<Container | null>(null);

export function WiringProvider({ container, children }: { container: Container; children: React.ReactNode }) {
  return <ContainerContext value={container}>{children}</ContainerContext>;
}

function useContainer(): Container {
  const container = useContext(ContainerContext);
  if (!container) throw new Error("[WiringProvider] Used outside the provider");
  return container;
}

export const useUseCases = () => useContainer().useCases;
export const useCoordinators = () => useContainer().coordinators;
export const useDataChangeSignal = () => useContainer().dataChangeSignal;
```

`useContainer` is not exported. The container never changes for the provider's lifetime, so the instances these hooks return are stable in dependency arrays.

## 4. Who may import what

| Importer | May import from `src/wiring/` | Must never import |
|---|---|---|
| View | Nothing. It imports its ViewModel hook | Anything in `src/wiring/` |
| ViewModel | `useUseCases` | `useCoordinators`, `useDataChangeSignal`, `createContainer`, any `*.wiring.ts` |
| `src/query/` invalidation bridge | `useDataChangeSignal` | Everything else |
| App root View | `useCoordinators` | Everything else |
| `src/app/providers.tsx` | `createContainer`, `WiringProvider` | Any `*.wiring.ts` |
| Test helpers | `createContainer`, `WiringProvider`, the `DataSources` type | Any other `*.wiring.ts` |
| Wiring factories | Factories and types of the same layer or below | Factories of a higher layer |
| Use Case, Repository, Data Source, Service classes | Nothing | `src/wiring/` |

Client-only view state, such as a selected filter shared by two components, is not wiring. It lives next to its ViewModel in `*.state.ts`; see `stack/view-models.md`.

## 5. Reactivity

When a Use Case changes data, every ViewModel watching that chain must update without polling.

| Mechanism | Reacts? | Use for |
|---|---|---|
| `useMutation` `onSuccess` → `invalidateQueries` | Yes, for queries under that key | Changes made by this ViewModel |
| `DataChangeSignal` → `useDataChangeInvalidation` | Yes, for every query in the domain | Changes made anywhere, including Services |
| `useSyncExternalStore(useCase.subscribe, useCase.getSnapshot)` | Yes, on every publish | Use Case-held state such as a session |
| Selector hook from a `*.state.ts` store | Yes | Client-only view state |
| Calling a Use Case method in an effect or handler | No: one-shot | Actions |
| `refetchInterval`, manual `refetch()` loops | Forbidden | Never use them to compensate for a broken chain |

`useDataChangeInvalidation` lives in `src/query/`, is mounted once in `AppShell`, subscribes to the signal from `useDataChangeSignal()`, and calls `queryClient.invalidateQueries({ queryKey: [change.domain] })`.

## 6. Breaking dependency cycles

If a lower layer must cause higher-level work, it raises a signal and never imports the higher layer.

1. The **signal** is a dependency-free Contract in `src/contracts/signals/`, such as `SignOutSignal`, with `request(reason)` and `subscribe(listener)`.
2. The **coordinator** is a Service outside the cycle, such as `SignOutCoordinatorService`. It subscribes to the signal and calls the Use Case.
3. The **app root** starts the coordinator in an effect and stops it on unmount. Boot cannot start it, because the Use Case it calls may re-run boot.

A signal that a Data Source raises must exist before the Data Source does. `createContainer` calls `createSignals()` first and passes the result down, so each container owns its signals and no module-level constant is needed:

```ts
// signals.wiring.ts
export type Signals = { signOut: SignOutSignal; dataChange: DataChangeSignal };

export function createSignals(): Signals {
  return { signOut: new SignOutSignal(), dataChange: new DataChangeSignal() };
}
```

Moving a call into a helper or a closure does not remove the dependency. It still runs on behalf of its owner.

## Quick checklist

1. Every `new` of a role class is inside `src/wiring/`.
2. The `Container` type exposes no Repository, Data Source or non-coordinator Service.
3. No View imports from `src/wiring/`, and a ViewModel imports only `useUseCases`.
4. Every data-changing Use Case publishes a `DataChangeSignal`.
5. No polling stands in for a missing signal.
