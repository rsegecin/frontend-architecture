---
paths:
  - "**/components/**/use-*.ts"
  - "**/providers/**/use-*.ts"
  - "**/lib/stores/**"
---

# Next.js Domain Hooks: ViewModels and Client State

A ViewModel is a hook in `use-<component>.ts`, next to the component's `index.tsx`. It composes domain hooks, local state, validation, effects and navigation, and returns plain view state plus action functions. The View only renders.

## Shape

```ts
// src/components/product-form/use-product-form.ts
"use client";

import { useTranslations } from "next-intl";
import { useState } from "react";
import { createProductSchema } from "@/domain/product/dto/create-product.dto";
import { ProductQueries } from "@/domain/product/hooks";
import { useRouter } from "@/i18n/navigation";
import { useApiErrorMessage } from "@/lib/hooks/use-api-error-message";

export function useProductForm() {
  const t = useTranslations("product.form");
  const router = useRouter();
  const apiErrorMessage = useApiErrorMessage();
  const createProduct = ProductQueries.useCreateProduct();

  const [name, setName] = useState("");
  const [error, setError] = useState("");

  const submit = async () => {
    const parsed = createProductSchema.safeParse({ name: name.trim() });
    if (!parsed.success) {
      setError(t("invalidName"));
      return;
    }

    try {
      await createProduct.mutateAsync(parsed.data);
      router.push("/products");
    } catch (submitError) {
      setError(apiErrorMessage(submitError));
    }
  };

  return {
    name,
    error,
    isPending: createProduct.isPending,
    changeName: (value: string) => {
      setName(value);
      setError("");
    },
    onSubmit: (event: React.FormEvent<HTMLFormElement>) => {
      event.preventDefault();
      submit();
    },
  };
}
```

## Rules

- Reach data only through `<Module>Queries` hooks. Never import a module's `http/` folder, `httpClient` or `axios`.
- Validate input with the DTO's zod schema (`safeParse`) before calling a mutation, and map the failing field to an i18n message.
- Handle a screen's errors here: `try/catch` around `mutateAsync`, or the query's `isError` and `error`. The message comes from `useApiErrorMessage()` or an i18n key, never from `error.message`.
- Never handle the session-expired 401. `httpClient` already refreshes the token or logs out.
- Return plain values and action functions. The View never receives the raw query or mutation object.
- Keep a few fields in `useState`. Use react-hook-form when a form grows large.
- A function with more than one parameter takes a named-parameters object.

## Client state

| State | Where it lives |
|---|---|
| Data from the API | React Query cache, never a store |
| The signed-in user | `useAuthStore`, filled by `AuthProvider`. The only API data in a store |
| State of one component | `useState` |
| Client state shared by several components | One zustand store per file in `src/lib/stores/`, exporting `use<Name>` |

```ts
// src/lib/stores/filters.ts
import { create } from "zustand";

interface FiltersState {
  search: string;
  setSearch: (search: string) => void;
}

export const useFilters = create<FiltersState>((set) => ({
  search: "",
  setSearch: (search) => set({ search }),
}));
```

- Read one value per selector, so the component re-renders only when that value changes: `useFilters((state) => state.search)`.
- Outside React, such as in a query callback or in `logout()`, read or change the store with `use<Name>.getState()`.

## Persisted stores

A store that must survive a reload uses the `persist` middleware and validates what comes back from `localStorage` with zod. If the stored value fails the schema, the defaults stay.

```ts
const settingsSchema = z.object({ sidebarCollapsed: z.boolean() });

export const useSettings = create<SettingsState>()(
  persist(
    (set) => ({
      sidebarCollapsed: false,
      setSidebarCollapsed: (sidebarCollapsed) => set({ sidebarCollapsed }),
    }),
    {
      name: "app:settings",
      // localStorage is untrusted (older versions, manual edits).
      merge: (persisted, current) => {
        const parsed = settingsSchema.safeParse(persisted);
        return parsed.success ? { ...current, ...parsed.data } : current;
      },
    },
  ),
);
```
