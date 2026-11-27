---
paths:
  - "**/domain/**/entities/**"
  - "**/domain/**/dto/**"
---

# Next.js Domain Hooks: Contracts

Each business domain is a module in `src/domain/<module>/`, always with the same four folders. Types shared by every module live in `src/domain/shared/`.

```text
src/domain/product/
  entities/product.ts            # shape of what the API returns
  dto/create-product.dto.ts      # zod schema and type of what is sent
  dto/update-product.dto.ts
  dto/list-products.dto.ts       # list filters
  http/product.service.ts        # see stack/repositories.md
  http/index.ts                  # the module's service registry
  hooks/product.queries.ts       # see stack/use-cases.md
  hooks/index.ts
src/domain/shared/
  entities/api-error.ts          # ApiError, ApiErrorCode
  entities/message.ts            # Success, Message: bodies of responses without a resource
  entities/pagination.ts         # Paginated<T>, PaginationMeta
  dto/pagination.dto.ts          # paginationSchema
```

## Entities

TypeScript interfaces in the shape of the API response, with no logic and no classes.

```ts
export interface Product {
  id: number;
  name: string;
  price: number;
  status: "ACTIVE" | "ARCHIVED";
  createdAt: string;
}
```

- Closed sets are literal unions.
- Dates stay `string`, as the API sends them. Format them in the View.
- Responses are not parsed at runtime. The interface is a trusted description of the API.
- A paginated response is `Paginated<T>`: `{ data: T[]; meta: PaginationMeta }`.

## DTOs

One file per write operation, and one per list filter. Each exports a zod schema and the type inferred from it, so the validation and the type never drift apart.

```ts
import { z } from "zod";

export const createProductSchema = z.object({
  name: z.string().min(2),
  price: z.number().positive(),
});

export type CreateProductDto = z.infer<typeof createProductSchema>;
```

- The schema is camelCase with a `Schema` suffix (`createProductSchema`). The type is PascalCase with a `Dto` suffix (`CreateProductDto`).
- Never write a DTO type by hand next to its schema.
- List filters extend `paginationSchema`, and replace `sortBy` with the route's enum:

  ```ts
  export const listProductsSchema = paginationSchema.extend({
    sortBy: z.enum(["name", "createdAt"]).optional(),
    query: z.string().optional(),
  });
  ```

- A sub-schema shared by several DTOs is exported from the DTO file that owns it, such as a `sessionDeviceSchema` used by every route that opens a session.

## API errors

`src/domain/shared/entities/api-error.ts` describes the API's error body:

```ts
export interface ApiError {
  error: {
    category: ApiErrorCategory;
    code: ApiErrorCode;
    statusCode: number;
    message: string;
    traceId: string;
    timestamp: number;
    details?: ErrorDetail[];
  };
}
```

- `ApiErrorCode` is a union of every code the API returns, such as `"ERR_INCORRECT_CREDENTIALS"`.
- Each code's message lives in `public/locales/<locale>/errors.json`, under `apiErrors.<code>`. `apiErrors.default` covers codes without a translation.
- A new code goes into the union and into every locale.

## Rules

- Contracts import only `zod` and other Contracts. No React, axios, service or hook.
- Domain services import them with `import type`.
