# Stack Profile: React + Next.js + Zustand

React 19 with Next.js 16 App Router and TypeScript 5: TanStack Query 5 for ViewModels, Zustand 5 for client-only view state, a plain container in React Context for dependency wiring, zod 4 for Contracts, and Vitest with React Testing Library for tests.

## Role map

Paths in the last column are relative to `.claude/rules/frontend-architecture/`.

| Core role | Concrete form | Location | Read first |
|---|---|---|---|
| View | Client Component named `*Screen`, `*Dialog`, or a component | `src/screens/<group>/`, `src/components/<role>/` | `stack/views.md` |
| ViewModel | Hook named `use<View>ViewModel`, built on TanStack Query and Zustand | Next to its View, `*.view-model.ts` | `stack/view-models.md` |
| Local UI state | React hooks, react-hook-form, then `*.controller.ts` | In or next to its View | `stack/views.md` |
| Use Case | Plain class named `*UseCase` | `src/use-cases/<domain>/*.use-case.ts` | `stack/use-cases.md` |
| Repository | Plain class named `*Repository`, plus interface | `src/repositories/api/`, `src/repositories/local/<entity>/` | `stack/repositories.md` |
| Data Source | Plain class plus interface | `src/data-sources/<capability>/` | `stack/data-sources.md` |
| Service | Plain class named `*Service` with `start()` and `stop()` | `src/services/<capability>/*.service.ts` | `stack/services.md` |
| Util | Pure functions or a static class, no wiring entry | `src/utils/<area>/` | `core/logic-placement.md` |
| Contracts | zod schemas, inferred types, enums, error classes | `src/contracts/` | `stack/data-models.md` |

Use Cases, Repositories, Data Sources and Services import no React, Zustand or TanStack Query code. Only `src/wiring/` creates them; see `stack/wiring.md`. App startup follows `stack/app-boot.md`, and tests follow `stack/tests.md`. Before adding a dependency, Read `stack/packages-by-layer.md`.

## Next.js boundaries

- A View that reads app or user data is a Client Component (`"use client"`), because ViewModels use hooks.
- Server Components are limited to layouts, metadata, and static content. They never call a Use Case, Repository or Data Source, so no data path skips the chain.
- A routed `page.tsx` is thin: it renders one `*Screen` and passes route params as props.

## Naming

Files are kebab-case. A ViewModel, its state file, and its controller mirror the View's file name, in the same directory.

| View file | ViewModel file | ViewModel hook | State file, when needed | Controller file, when needed |
|---|---|---|---|---|
| `order-form-screen.tsx` | `order-form-screen.view-model.ts` | `useOrderFormScreenViewModel` | `order-form-screen.state.ts` | `order-form-screen.controller.ts` |
| `depth-input-dialog.tsx` | `depth-input-dialog.view-model.ts` | `useDepthInputDialogViewModel` | `depth-input-dialog.state.ts` | |
| `sorting-filter.tsx` | `sorting-filter.view-model.ts` | `useSortingFilterViewModel` | | |

A screen ViewModel keeps the `-screen` infix: `order-form-screen.view-model.ts`, never `order-form.view-model.ts`.

| Role | File suffix | Class or export |
|---|---|---|
| Use Case | `.use-case.ts` | `ExportOrdersUseCase` |
| Repository | `.repository.ts`, interface `.repository.int.ts` | `ApiOrdersRepository` implements `OrdersRepository` |
| Data Source | `.data-source.ts`, interface `.data-source.int.ts` | `AxiosHttpClient` implements `HttpClient` |
| Service | `.service.ts` | `OrdersSyncService` |
| DTO | `.dto.ts` | `OrderDtoSchema` and `type OrderDto` |
| Wiring | `.wiring.ts` | `createOrdersUseCases` |

## Directory map

```text
src/
  app/            # Next.js routes: layout.tsx, providers.tsx, **/page.tsx (thin)
  contracts/      # Contracts: zod schemas, models, enums, errors, signals, mappers
  data-sources/
  repositories/
  use-cases/
  services/
  wiring/         # container factory and its React Context
  query/          # QueryClient factory, query keys, invalidation bridge
  screens/        # screen groups, with colocated ViewModels
  components/     # shared UI
  hooks/          # custom UI hooks
  i18n/
  utils/
  config/         # constants, endpoints
test/
  helpers/        # shared fakes and render helpers
  setup.ts        # Vitest setup file
__mocks__/
  zustand.ts      # resets view-state stores after each test
```

Create a directory only when its first file is added.

## Commands

| Task | Command |
|---|---|
| Generate code | none |
| Analyze | `npx tsc --noEmit` and `npm run lint` |
| Test | `npx vitest run` |
