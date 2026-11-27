# Stack Profile: Next.js + Domain Hooks

React 19 with Next.js 16 App Router and strict TypeScript: TanStack Query 5 hooks grouped by domain module, static service classes per domain module over one axios client, zustand 5 for shared client state, zod 4 for outgoing DTOs, next-intl 4 for localized routes and copy, Tailwind CSS 4 with shadcn/ui, and Biome for lint and format. No test tool yet.

This profile has fewer files than `stacks/react-zustand`: a domain hook plays the Use Case, a static registry in each domain module replaces the container, and the API owns the business rules. It fits API-backed dashboards and back offices, where the front end mostly reads, writes and caches API resources.

## Role map

Paths in the last column are relative to `.claude/rules/frontend-architecture/`.

| Core role | Concrete form | Location | Read first |
|---|---|---|---|
| View | Route `page.tsx`, or a component file in kebab-case with a PascalCase named export | `src/app/(frontend)/[locale]/`, `src/components/`, `src/providers/` | `stack/views.md` |
| ViewModel | Hook `use<Component>` in `use-<component>.ts`, next to the component's `index.tsx`. Shared client state is a zustand store | Component folder; stores in `src/lib/stores/` | `stack/view-models.md` |
| Local UI state | `useState`, then react-hook-form for larger forms | In the ViewModel hook, or in the View when it has no ViewModel | `stack/view-models.md` |
| Use Case | Domain hook `use<Method>`, grouped in a `<Module>Queries` object | `src/domain/<module>/hooks/<module>.queries.ts` | `stack/use-cases.md` |
| Repository | Domain service: static class `<Resource>Service` extending `Service`, registered in the module's `Service` registry | `src/domain/<module>/http/<resource>.service.ts` | `stack/repositories.md` |
| Data Source | Configured module instances: `httpClient`, the `QueryClient`, token storage | `src/config/`, `src/lib/session.ts` | `stack/data-sources.md` |
| Service | None yet | Ask the user before adding one | `core/logic-placement.md` |
| Util | Pure functions, constants, reusable UI hooks | `src/lib/`, `src/lib/hooks/` | `core/logic-placement.md` |
| Contracts | Entity interfaces, DTO zod schemas | `src/domain/<module>/entities/`, `src/domain/<module>/dto/`, `src/domain/shared/` | `stack/data-models.md` |

App startup, routing and the session gate follow `stack/app-boot.md`. Before adding a dependency, Read `stack/packages-by-layer.md`.

In this stack, *service* names the Repository layer: `src/services/` holds the `Service` base class and the `Key` helper, and each module's `http/` holds its `<Resource>Service` classes and the `Service` registry. None of them is the core Service role.

## How this stack applies the core chain

```text
User path:  page or component -> use-<component>.ts -> <Module>Queries hook -> <Resource>Service -> httpClient -> API
```

- A component whose only logic is one domain hook call stays one file and reads that hook directly. The hook already returns view state (`data`, `isPending`, `mutateAsync`). Once the component holds state, validation, effects or more than one hook, it becomes a folder with a `use-<component>.ts` ViewModel.
- The domain hook is the only entry point to a service method. It owns the query key, the invalidation, and session side effects.
- Business rules live in the API. The front end checks input against the DTO schema before it calls a mutation.
- Views and ViewModels never import a module's `http/` folder, `@/config/axios-client` or `axios`.

## Where this stack departs from core

These are deliberate. Do not refactor them toward `core/` unless the user asks.

1. Use Cases are React hooks built on TanStack Query, not plain classes.
2. There are no interfaces and no container. Domain services are static classes, each module registers its own in `http/index.ts`, and `httpClient` is a module instance.
3. `AxiosError` crosses layers as the transport error. Only `useApiErrorMessage` and `src/config/query-client.ts` inspect it; other code treats errors as `unknown`.
4. The `httpClient` response interceptor calls `Service.auth.refreshSession` (from `@/domain/auth/http`) and `logout()` to renew or end the session. This is the only lower-to-higher call in the stack. Ask the user before adding another.
5. There is no boot Use Case. The `AuthProvider` in the `(private)` layout is the boot shell for signed-in pages.

## Naming

Files are kebab-case. Components, classes and types are PascalCase; variables and functions are camelCase; hooks start with `use`.

| Role | File | Export |
|---|---|---|
| Page | `src/app/(frontend)/[locale]/(private)/products/page.tsx` | default export `ProductsPage` |
| Component without logic | `src/components/delete-product-button.tsx` | `DeleteProductButton` |
| Component with logic | `src/components/product-form/index.tsx` | `ProductForm` |
| ViewModel | `src/components/product-form/use-product-form.ts` | `useProductForm` |
| Sub-component of the same flow | `product-form/price-input.tsx` and `product-form/use-price-input.ts` | `PriceInput`, `usePriceInput` |
| Domain hooks | `src/domain/product/hooks/product.queries.ts`, re-exported by `hooks/index.ts` | `ProductQueries` |
| Domain service | `src/domain/product/http/product.service.ts` | `ProductService`, registered as `Service.product` in `http/index.ts` |
| Entity | `src/domain/product/entities/product.ts` | `interface Product` |
| DTO | `src/domain/product/dto/create-product.dto.ts` | `createProductSchema`, `type CreateProductDto` |
| Store | `src/lib/stores/filters.ts` | `useFilters` |

A domain hook takes its service method's name with a `use` prefix: `getAllProducts` becomes `useGetAllProducts`.

## Directory map

```text
src/
  app/
    (frontend)/             # route group for pages, not part of the URL
      globals.css           # Tailwind config and theme tokens
      [locale]/             # every page sits under the locale
        layout.tsx          # validates the locale, mounts the providers
        (private)/          # pages that need a signed-in user
    api/                    # route handlers, no locale
  components/               # project components and their ViewModels
    ui/                     # shadcn components
  config/                   # httpClient and QueryClient
  domain/<module>/          # entities/, dto/, http/, hooks/
  domain/shared/            # pagination, API error shape
  i18n/                     # next-intl routing, navigation, request config
  lib/                      # Utils, constants, session.ts
    hooks/                  # reusable UI hooks
    stores/                 # zustand stores
  providers/                # React providers
  services/                 # Service base class and Key helper for domain services
  proxy.ts                  # Next 16 proxy, formerly middleware
public/locales/<locale>/    # messages.json and errors.json
```

`@/*` points to `src/*`. Import through the alias, never through `../` paths that leave the module. Create a directory only when its first file is added.

## Next.js 16

- `params` and `cookies()` are Promises.
- The middleware file is `src/proxy.ts`.
- `next lint` does not exist. Biome lints and formats.

## Commands

| Task | Command |
|---|---|
| Generate code | none. Add a base component with `npx shadcn@latest add <component>` |
| Analyze | `npm run lint:strict` and `npm run typecheck` |
| Build | `npm run build` |
| Format | `npm run format` |
| Test | none yet |
