---
paths:
  - "**/domain/**/http/**"
  - "**/*.service.ts"
  - "**/services/**"
---

# Next.js Domain Hooks: Domain Services (Repository role)

Every HTTP call lives in the `http/` folder of its domain module, one `<resource>.service.ts` per API resource, registered in the module's `http/index.ts`. No other folder imports `httpClient`.

These domain services play the core Repository role. They are not the core Service role.

## Shape

```ts
// src/domain/product/http/product.service.ts
import { httpClient } from "@/config/axios-client";
import type { Paginated } from "@/domain/shared/entities/pagination";
import { HttpMethods, Service } from "@/services";
import type { CreateProductDto } from "../dto/create-product.dto";
import type { ListProductsDto } from "../dto/list-products.dto";
import type { Product } from "../entities/product";

export class ProductService extends Service {
  public static key = "@product";
  public static defaultPath = "/products";

  public static async getAllProducts(): Promise<Product[]> {
    const method = HttpMethods.GET;
    const path = Service.endpoint(ProductService.defaultPath, "");
    const { data } = await httpClient[method](path);
    return data;
  }

  public static async listProducts(query?: ListProductsDto): Promise<Paginated<Product>> {
    const method = HttpMethods.GET;
    const path = Service.endpoint(ProductService.defaultPath, "/search");
    const { data } = await httpClient[method](path, { params: query });
    return data;
  }

  public static async createProduct(dto: CreateProductDto): Promise<Product> {
    const method = HttpMethods.POST;
    const path = Service.endpoint(ProductService.defaultPath, "");
    const { data } = await httpClient[method](path, dto);
    return data;
  }

  public static async deleteProduct(id: number): Promise<void> {
    const method = HttpMethods.DELETE;
    const path = Service.endpoint(ProductService.defaultPath, `/${id}`);
    await httpClient[method](path);
  }
}
```

- `key` is the query-key prefix, in the form `@<resource>`. It must be unique across every service in the app, not only in its module.
- `defaultPath` is the resource's base path in the API.
- Each method picks an `HttpMethods` value, builds the path with `Service.endpoint(defaultPath, suffix)`, calls `httpClient[method]`, and declares its return type.
- GET filters go in `{ params: query }`. Bodies go as the second argument.
- A route outside `defaultPath` uses `Service.endpoint("", "/verify-email")`, with a comment saying why.
- For multipart bodies, pass `Service.toFormData(dto)`. It skips `undefined` and `null` fields and expands a `FileList` into one field per file.
- The module's own entities and DTOs are imported with relative paths (`../dto/...`, `../entities/...`). Types from another module or from `domain/shared` come through `@/domain/...`.

## Module registry

Each module's `http/index.ts` registers its services in a static class named `Service`:

```ts
// src/domain/product/http/index.ts
import { ProductService } from "./product.service";

export class Service {
  static product = ProductService;
}
```

- The registry has the same name as the base class in `@/services`. Inside a module's `hooks/`, `Service` is always this registry, imported from `../http`.
- A module with several resources registers each one under its name in camelCase, such as `Service.customField` and `Service.customFieldOption`.
- Only the module's own hooks import its registry. The one exception is `src/config/axios-client.ts`, which imports `@/domain/auth/http` for the token refresh; see `stack/data-sources.md`.

## `src/services/`

The shared base of every domain service. It is not the core Service role.

| Export | Job |
|---|---|
| `Service` | Base class: `key`, `defaultPath`, `endpoint()`, `toFormData()` |
| `HttpMethods` | Enum of the axios method names |
| `Key(ctor, method)` | Builds a query key: `"<ctor.key>:<method>"` |

## Rules

- Methods are static, and a service holds no state. Biome's `noStaticOnlyClass` is off for this reason.
- Refer to the class by name (`ProductService.defaultPath`), never `this`, because domain hooks pass methods unbound as `queryFn` and `mutationFn`.
- Import entities and DTOs only as types, with `import type`.
- Return the typed `data` as the API sends it. Responses are not parsed at runtime. The entity interface is the contract.
- Make no business decisions, hold no cache, and import no React, TanStack Query or zustand.
- Never catch errors here. They reach the ViewModel unchanged.
