---
paths:
  - "**/domain/**/hooks/**"
  - "**/*.queries.ts"
---

# Next.js Domain Hooks: Domain Hooks (Use Case role)

Each module's `hooks/<module>.queries.ts` holds one hook per service method, grouped and exported as `<Module>Queries`. `hooks/index.ts` re-exports the file, and components import from `@/domain/<module>/hooks`.

## Shape

```ts
// src/domain/product/hooks/product.queries.ts
"use client";

import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import type { ListProductsDto } from "@/domain/product/dto/list-products.dto";
import { Key } from "@/services";
import { Service } from "../http";

function useGetAllProducts() {
  return useQuery({
    queryKey: [Key(Service.product, "getAllProducts")],
    queryFn: Service.product.getAllProducts,
  });
}

function useListProducts(query?: ListProductsDto) {
  return useQuery({
    queryKey: [Key(Service.product, "listProducts"), query],
    queryFn: () => Service.product.listProducts(query),
  });
}

function useGetProductById(id: number) {
  return useQuery({
    queryKey: [Key(Service.product, "getProductById"), id],
    queryFn: () => Service.product.getProductById(id),
    enabled: !!id,
  });
}

function useCreateProduct() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationKey: [Key(Service.product, "createProduct")],
    mutationFn: Service.product.createProduct,
    onSuccess: () => {
      queryClient.invalidateQueries({
        queryKey: [Key(Service.product, "listProducts")],
      });
    },
  });
}

export const ProductQueries = {
  useGetAllProducts,
  useListProducts,
  useGetProductById,
  useCreateProduct,
};
```

```ts
// src/domain/product/hooks/index.ts
export * from "./product.queries";
```

## Query keys

- Build every key with `Key(Service.<resource>, "<method>")`. It returns `"<key>:<method>"`, such as `"@product:getAllProducts"`. Never write key strings by hand.
- The `Key` string is always the first element. Parameters follow it: `[Key(...), id]` or `[Key(...), query]`.
- TanStack matches keys by prefix, so invalidating `[Key(Service.product, "listProducts")]` refreshes every page and filter of that list.

## Passing the service method

Pass the method itself only when its parameters match what TanStack passes. Otherwise wrap it in an arrow function.

| Hook | Pass the method directly when | Wrap it when |
|---|---|---|
| `queryFn` | It takes no parameters. TanStack calls `queryFn` with its query context | It takes any parameter, even an optional one: `() => Service.user.listUsers(query)` |
| `mutationFn` | Its only parameter is the mutation variable | It takes no parameters: `() => Service.auth.logout()` |

This works because service methods never use `this`; see `stack/repositories.md`.

## Rules

- The file starts with `"use client"`.
- Every `queryFn` and `mutationFn` calls one `Service.<resource>` method, through the module's own registry imported from `../http`.
- A query that depends on a parameter uses `enabled: !!param`. A hook the caller may disable takes a named option: `useGetCurrentUser({ enabled })`.
- Every mutation invalidates the queries it affects in `onSuccess`. When it affects several, use a local helper:

  ```ts
  function invalidate({
    queryClient,
    methods,
  }: {
    queryClient: QueryClient;
    methods: (keyof typeof Service.product)[];
  }): void {
    for (const method of methods) {
      queryClient.invalidateQueries({ queryKey: [Key(Service.product, method)] });
    }
  }
  ```

- A mutation that opens a session, because it returns `SessionTokens`, uses `onSuccess: saveSession` from `@/lib/session`.
- Get the client with `useQueryClient()`. Never create a `QueryClient`.
- No URLs, no `httpClient`, no JSX, no translated text and no `try/catch` here. Errors reach the ViewModel, which maps them to a message.
- Never use `refetchInterval` or manual `refetch()` loops to keep data fresh. Invalidate instead.
