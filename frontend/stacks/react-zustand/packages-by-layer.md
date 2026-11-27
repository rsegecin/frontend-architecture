---
paths:
  - "**/package.json"
---

# React: Packages by Layer

This guide lists the packages this stack uses to implement each layer in `core/layers.md`. Read it before adding a dependency. It leaves out Util packages (such as JWT, phone or date helpers) and dev tooling (linters, formatters, type definitions, test runners, CSS build tools) on purpose: they implement no layer, and a project picks them freely as long as Utils stay stateless and free of business rules.

## Layer legend

```text
User path:    View -> ViewModel -> Use Case -> Repository -> Data Source
System path:  Service -> Use Case -> Repository -> Data Source
```

| Layer | Job |
|---|---|
| **View** | Renders state and forwards user input. |
| **Local UI state** | Ephemeral state only: inputs, scroll, animation, layout. |
| **ViewModel** | View-specific state and actions for one View, bridged to Use Cases. |
| **Use Case** | Business orchestration. Plain TypeScript. |
| **Repository** | The data boundary. Plain TypeScript. |
| **Data Source** | Wraps one external system. |
| **Service** | Long-lived system actor. Plain TypeScript. |
| **Contracts** | Models, DTOs, mappers, enums and typed errors that depend on nothing. |
| **Dependency wiring** | Outside the chain. Creates every instance and hands each layer its dependencies. |

## Packages

Grouped in layer order, from the bottom of the chain up.

| Package | Major version | Layer | What it is | Why this layer |
|---|---|---|---|---|
| zod | 4 | Contracts | Schema declaration and parsing library with inferred TypeScript types. | DTOs are zod schemas, and Repositories parse payloads with them. It is a pure library with no I/O, so Contracts may import it. |
| axios | 1 | Data Source | Promise-based HTTP client with interceptors, timeouts and cancellation. | Wraps one external system: HTTP. Only the HTTP Data Source imports it, and it converts `AxiosError` into Contract errors. |
| firebase | 12 | Data Source | Google Firebase SDK. This stack uses Firebase Auth for sign-in and ID tokens. | External SDK. Its calls, `User` objects and error codes stay inside the auth Data Source. |
| idb | 8 | Data Source (optional) | Small promise wrapper over IndexedDB. | Wraps local persistence, one external system. Add it only when a project needs structured local storage. |
| @tanstack/react-query | 5 | ViewModel | Async server-state library: caching, deduplication, retries and invalidation through `useQuery` and `useMutation`. | ViewModel hooks call Use Cases through `queryFn` and `mutationFn`, and invalidation keeps every watching View in sync. |
| zustand | 5 | ViewModel | Small store library for React (`create`, selector hooks). | Holds client-only view state in private stores in `*.state.ts`. Shared view state, such as a selected tab, is view-specific. Server data stays in the query cache, and wiring never uses it. |
| i18next | 26 | ViewModel (presentation) | Framework-agnostic translation engine: resources, interpolation, plurals. | ViewModels map state and errors to translation keys. Use Cases never hold view-specific strings. |
| react | 19 | View + Local UI state | UI library for components and hooks. | Components are Views; `useState`, `useReducer` and `useRef` hold ephemeral local UI state. |
| react-dom | 19 | View | Renders React into the browser DOM. | Pure rendering concern. |
| next | 16 | View (app root, routing) | React framework with the App Router: layouts, pages and file-based routing. | The root layout and providers form the app root that hosts the boot shell and app shell. Pages stay thin and render one screen. |
| react-i18next | 17 | View | React bindings for i18next (`useTranslation`, `I18nextProvider`). | Views translate the keys their ViewModel returns. |
| react-hook-form | 7 | Local UI state (optional) | Form state and field registration with uncontrolled inputs. | Field values, dirty and touched flags are ephemeral UI state. Submitting goes through the ViewModel. |
| @hookform/resolvers | 5 | Local UI state (optional) | Adapters that let react-hook-form validate with a zod schema. | Reuses the Contract schema for field-level checks. Business preconditions still live in the Use Case. |
| react-imask | 7 | Local UI state (optional) | Input-masking component (`IMaskInput`) for formatted fields such as tax ids or phone numbers. | Masking is a component concern over the field's ephemeral state. |

## Layers with no package

**Use Case**, **Repository** and **Service** need no third-party package. They are plain TypeScript classes that receive their dependencies through the constructor:

- Use Cases depend on Repository interfaces, signals and Utils.
- Repositories depend on Data Source interfaces and Contracts.
- Services depend on Use Cases and on the browser events that trigger them.

Keeping these layers free of SDKs and React code lets them run in plain unit tests, and lets Data Sources be swapped without touching business logic.

**Dependency wiring** needs no package either. `src/wiring/` is plain factory functions that call constructors, and React Context from `react` hands the result to hooks. See `stack/wiring.md`.

## Avoid with this stack

| Package | Why |
|---|---|
| swr | Overlaps TanStack Query. Two server-state caches can't share invalidation. |
| @reduxjs/toolkit, RTK Query | Overlaps both Zustand and TanStack Query, and pulls business logic into reducers and thunks. |
| jotai | Overlaps Zustand for view state, and would add a second way to build instances next to the container. |
| inversify, tsyringe | Need decorators and `reflect-metadata`. The container is plain constructor calls and needs no DI library. |
| class-transformer, class-validator, reflect-metadata | Replaced by zod. They need decorators and a metadata polyfill, and they duplicate the schema as a class. |
| next-auth, other server-session libraries | Moves authentication into server code that bypasses the Data Source layer. Add one only with a stack-profile change that maps it to a role. |

To add a package that is not listed here, first name the layer it belongs to. If it implements a layer, add a row to this file. If it fits no layer and is not a Util or dev tool, raise it with the user before installing it.
