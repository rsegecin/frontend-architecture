---
paths:
  - "**/*-screen.tsx"
  - "**/*-dialog.tsx"
  - "**/screens/**"
  - "**/components/**"
  - "**/app/**"
---

# React: Views

## Choosing a component type

| Component needs | Use |
|---|---|
| No app state, no local state, no hooks | Server Component or plain function component |
| A ViewModel | Client Component (`"use client"`) that calls `use<View>ViewModel()` |
| A ViewModel and local UI state | Client Component with both the ViewModel hook and React hooks |
| Local UI state only | Client Component with React hooks |

Server Components never read app data. They render layouts, metadata and static content only.

## Routes stay thin

A `page.tsx` renders one screen and passes route params as props. It holds no state and calls no hooks beyond reading params.

```tsx
// src/app/(dashboard)/orders/[orderId]/page.tsx
import { OrderDetailScreen } from "@/screens/orders/order-detail-screen";

export default async function Page({ params }: { params: Promise<{ orderId: string }> }) {
  const { orderId } = await params;
  return <OrderDetailScreen orderId={orderId} />;
}
```

## Reading a ViewModel

```tsx
// src/screens/settings/settings-screen.tsx
"use client";

import { useTranslation } from "react-i18next";
import { useSettingsScreenViewModel } from "./settings-screen.view-model";

export function SettingsScreen() {
  const { t } = useTranslation();
  const vm = useSettingsScreenViewModel();

  return (
    <section>
      <button disabled={!vm.isExportAvailable || vm.isExporting} onClick={vm.exportOrders}>
        {t("settings.export")}
      </button>
      {vm.errorKey ? <p role="alert">{t(vm.errorKey)}</p> : null}
    </section>
  );
}
```

A View renders what its ViewModel returns and calls its action methods. It never imports from `src/wiring/`, `src/use-cases/`, `src/repositories/`, `src/data-sources/` or `src/services/`, and never calls `useQuery`, `useMutation` or `useQueryClient` itself. It shows translated keys from view state, never raw error text.

## Local UI state

Default to React hooks: `useState`, `useReducer`, `useRef`, `useId`. Use react-hook-form for form fields, with the field schema from Contracts through `@hookform/resolvers/zod`. Move UI logic into a colocated `*.controller.ts` hook only when it carries logic of its own, such as keyboard navigation or debouncing. Custom reusable UI hooks live in `src/hooks/`.

```text
Bad:  useState for an order list loaded from the API     # server data belongs in the ViewModel
Good: useState for whether the filter panel is open
```

Never put UI-control objects (refs, timers, form instances) in a ViewModel, and never put business rules in a controller.

## Ownership paths

Shared UI lives in `src/components/`:

```text
src/components/
  app/      # app root and shells, see stack/app-boot.md
  dialogs/  # shared dialog building blocks
  inputs/
  layout/
```

Screen-group UI lives under `src/screens/<group>/components/`:

```text
src/screens/authentication/components/
src/screens/dashboard/components/
  shell/    # subfolders once a real cluster exists
```

```text
Bad:  src/components/authentication/pin-input.tsx        # domain component in shared UI
Good: src/screens/authentication/components/pin-input.tsx

Good: src/components/layout/app-scaffold.tsx             # generic, even with one user today
```

Every screen file ends in `-screen.tsx`, and every dialog in `-dialog.tsx`. Group screen directories by user journey, never one flat folder.
