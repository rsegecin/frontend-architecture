---
paths:
  - "**/package.json"
---

# Next.js Domain Hooks: Packages by Layer

This guide lists the packages this stack uses for each layer in `core/layers.md`, as `stack/stack-profile.md` maps them. Read it before adding a dependency. Dev tooling (Biome, husky, commitlint, type definitions, Tailwind build plugins) implements no layer and is left out.

## Packages

Grouped in layer order, from the bottom of the chain up.

| Package | Major version | Layer | Why this layer |
|---|---|---|---|
| zod | 4 | Contracts | DTOs are zod schemas with inferred types. Persisted stores also validate `localStorage` with it |
| axios | 1 | Data Source | Only `src/config/axios-client.ts` creates the instance, and only domain services in `src/domain/<module>/http/` call it. Its interceptors hold the auth header and the token refresh |
| @tanstack/react-query | 5 | Use Case | Domain hooks wrap each service method in `useQuery` or `useMutation`, and invalidation keeps every View in sync |
| @tanstack/react-query-devtools | 5 | Use Case tooling | Mounted once in `QueryProvider` |
| zustand | 5 | ViewModel | Client state shared by several components, one store per file in `src/lib/stores/`. API data stays in the query cache |
| react-hook-form | 7 | Local UI state | Field state for larger forms. Submitting still goes through the ViewModel |
| next | 16 | View, routing | App Router: layouts, pages, route groups and route handlers |
| next-intl | 4 | View, routing | Locale routing, the proxy middleware, translated copy and locale-aware navigation |
| next-themes | 0 | View | Light and dark theme through `data-theme` on `<html>` |
| react, react-dom | 19 | View, Local UI state | Components are Views, and `useState` holds local state |
| radix-ui, class-variance-authority, clsx, tailwind-merge, cn, lucide-react | various | View | What the shadcn/ui components in `src/components/ui/` are built on |

## Layers with no package

- **Repository** needs no package beyond the `httpClient` instance. Each domain service is a static class over the `Service` base in `src/services/`.
- **Dependency wiring** needs none. Each module's `Service` registry in `http/index.ts` is a static class.

## Avoid with this stack

| Package | Why |
|---|---|
| swr, RTK Query | A second server-state cache. It cannot share invalidation with TanStack Query |
| @reduxjs/toolkit, jotai, valtio | A second client-state library next to zustand |
| fetch wrappers such as ky or ofetch | A second HTTP client skips the `httpClient` interceptors, so it loses the auth header and the token refresh |
| i18next, react-i18next | next-intl already owns translation and locale routing |
| next-auth, cookie session libraries | The session lives in `localStorage` behind `src/lib/session.ts`. Changing that is an architecture change |
| class-validator, class-transformer | Replaced by zod |
| eslint, prettier | Biome lints and formats |

To add a package that is not listed here, first name the layer it belongs to. If it implements a layer, add a row to this file. If it fits no layer and is not a Util or dev tool, raise it with the user before installing it.
