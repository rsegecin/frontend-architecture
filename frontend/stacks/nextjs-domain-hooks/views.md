---
paths:
  - "**/app/**/page.tsx"
  - "**/components/**/*.tsx"
---

# Next.js Domain Hooks: Views

## Choosing a component type

| Component needs | Use |
|---|---|
| No hooks | Server Component, the default |
| Any hook: state, effect, React Query, router, store | Client Component: the file starts with `"use client"` |

`useTranslations` works in Client Components and in non-async Server Components. An `async` Server Component uses `await getTranslations()` from `next-intl/server`.

## Pages

A page is a folder under `[locale]`. A page that needs a signed-in user goes under `[locale]/(private)/`, whose layout mounts the session gate.

```text
src/app/(frontend)/[locale]/(private)/products/page.tsx        ->  /en/products
src/app/(frontend)/[locale]/(private)/products/[id]/page.tsx   ->  /en/products/42
```

```tsx
export default async function ProductPage({
  params,
}: {
  params: Promise<{ locale: string; id: string }>;
}) {
  const { id } = await params;
  return <ProductDetail id={id} />;
}
```

A page lays out components and passes route params down. It reads data only through domain hooks, like any component.

## Component without logic

A component whose only logic is one domain hook stays one file and reads that hook directly.

```tsx
"use client";

import { useTranslations } from "next-intl";
import { Button } from "@/components/ui/button";
import { ProductQueries } from "@/domain/product/hooks";

export function DeleteProductButton({ id }: { id: number }) {
  const t = useTranslations();
  const { mutateAsync, isPending } = ProductQueries.useDeleteProduct();

  const onDelete = async () => {
    try {
      await mutateAsync(id);
      alert(t("product.delete.success"));
    } catch {
      alert(t("product.delete.error"));
    }
  };

  return (
    <Button variant="destructive" disabled={isPending} onClick={onDelete}>
      {t("product.delete.label")}
    </Button>
  );
}
```

## Component with logic

A component with its own state, validation, effects or several hooks becomes a folder. `index.tsx` only renders what `use-<name>.ts` returns; see `stack/view-models.md`.

```tsx
// src/components/product-form/index.tsx
"use client";

export function ProductForm() {
  const t = useTranslations("product.form");
  const form = useProductForm();

  return (
    <form noValidate onSubmit={form.onSubmit}>
      <Label htmlFor="product-name">{t("name")}</Label>
      <Input
        id="product-name"
        value={form.name}
        onChange={(event) => form.changeName(event.target.value)}
      />
      {form.error && <p role="alert">{form.error}</p>}
      <Button type="submit" disabled={form.isPending}>
        {t("submit")}
      </Button>
    </form>
  );
}
```

A sub-component of the same flow sits in the folder with its own pair, such as `price-input.tsx` and `use-price-input.ts`, and talks to the parent through props and callbacks (`onBack`, `onSuccess`).

## Text and navigation

- No visible text is hard-coded in JSX, including button labels, loading states and alerts. Every string goes to `public/locales/<locale>/messages.json`, in every locale, with keys grouped by feature.
- Never show raw error text. API errors go through `useApiErrorMessage()`; other failures use an i18n key.
- Navigate with `Link`, `useRouter`, `redirect`, `usePathname` and `getPathname` from `@/i18n/navigation`, never from `next/link` or `next/navigation`, so the locale stays in the URL.
- Route handlers under `/api` have no locale. Reach them with `window.location.assign("/api/<route>")`, because the next-intl router would add a locale prefix.
- Switch language with `router.replace(pathname, { locale })`.

## Styling

- Before writing a base component (button, input, dialog), check shadcn and add it with `npx shadcn@latest add <component>`. It lands in `src/components/ui/`, and those files are project code you may edit.
- Project components are built on `src/components/ui/`, and merge classes with `cn()` from `@/lib/utils`.
- Colors use theme tokens (`bg-background`, `text-foreground`, `bg-muted`, `bg-primary`, `border-border`), never fixed palette classes such as `bg-gray-100`. A new color becomes a token in `:root`, in `[data-theme="dark"]`, and in `@theme inline` in `globals.css`.
- Text styles use `type-*` utilities, not `text-*`: `tailwind-merge` reads `text-h1` as a color and drops it next to `text-foreground`.
- next-themes sets `data-theme` on `<html>`, and the `dark:` variant follows it. A page without a dark theme goes in the light-only list of the theme provider.

## Rules

- A View never imports a module's `http/` folder, `@/config/*` or `axios`.
- A View never creates a `QueryClient`, and never writes server data into a store.
- Images go through `next/image`, and decorative ones get `alt=""`.
