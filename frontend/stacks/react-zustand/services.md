---
paths:
  - "**/services/**"
  - "**/*.service.ts"
---

# React: Services

A Service is a plain class named `*Service`, in `src/services/<capability>/*.service.ts`. It exposes `start()` and `stop()`. Add a sibling `*.service.int.ts` when other code depends on its surface.

## Shape

```ts
// src/services/orders-sync/orders-sync.service.ts
import type { SyncOrdersUseCase } from "@/use-cases/orders/sync-orders.use-case";

export class OrdersSyncService {
  private timer: ReturnType<typeof setInterval> | null = null;
  private readonly onOnline = () => void this.sync();

  constructor(private readonly syncOrders: SyncOrdersUseCase) {}

  start(): void {
    if (this.timer) return;
    this.timer = setInterval(() => void this.sync(), 5 * 60_000);
    window.addEventListener("online", this.onOnline);
  }

  stop(): void {
    if (this.timer) clearInterval(this.timer);
    this.timer = null;
    window.removeEventListener("online", this.onOnline);
  }

  private async sync(): Promise<void> {
    try {
      await this.syncOrders.execute();
    } catch (error) {
      console.error("[OrdersSyncService] Sync failed", error);
    }
  }
}
```

`start()` is idempotent, and `stop()` releases every timer and listener that `start()` created.

## Coordinators

A coordinator is a Service that watches a signal and calls a Use Case, to break a dependency cycle. See `stack/wiring.md`.

```ts
// src/services/session/sign-out.coordinator.service.ts
export class SignOutCoordinatorService {
  private unsubscribe: (() => void) | null = null;

  constructor(
    private readonly signal: SignOutSignal,
    private readonly signOut: SignOutUseCase,
  ) {}

  start(): void {
    this.unsubscribe ??= this.signal.subscribe(() => void this.signOut.execute());
  }

  stop(): void {
    this.unsubscribe?.();
    this.unsubscribe = null;
  }
}
```

The app root starts and stops coordinators, because boot cannot. See `stack/app-boot.md`.

## Rules

- Do the work through Use Cases. Never take a Repository or Data Source in the constructor.
- Subscribe directly only to the external event that triggers the Service, such as `online`, `visibilitychange` or a device stream. What it does in response goes through a Use Case.
- Hold no UI state, and import no React, Zustand or TanStack Query.
- The boot Use Case starts and stops Services. Views and ViewModels never call `start()` or `stop()`, except the app root for coordinators.
- To show a Service's state in the UI, expose it from a Use Case with `subscribe`/`getSnapshot`, and let the ViewModel read that.
- A one-shot action is not a Service. If nothing needs to run over time, write a Use Case or a Util.
