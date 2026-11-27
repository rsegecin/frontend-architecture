---
paths:
  - "**/app/layout.tsx"
  - "**/app/providers.tsx"
  - "**/app-root.tsx"
  - "**/boot/**"
  - "**/*-init.use-case.ts"
---

# React: App Boot

`core/cross-cutting.md` defines the boot rules. This file gives their Next.js App Router form.

## Tree

```text
src/app/layout.tsx          # Server Component: html, body, metadata
  src/app/providers.tsx     # Client: WiringProvider + QueryClientProvider
    AppRoot                 # keeps coordinators alive, picks the shell
      BootShell             # phase 1, until the first boot succeeds
      AppShell              # phase 2, wraps the routed children
```

```text
src/components/app/
  app-root.tsx
  boot/
    initial-boot-complete.state.ts
    phase-1/
      boot-shell.tsx
      boot-shell.view-model.ts
    phase-2/
      app-shell.tsx
      app-shell.view-model.ts
      overlays/
```

## Providers

```tsx
// src/app/providers.tsx
"use client";

export function Providers({ children }: { children: React.ReactNode }) {
  const [container] = useState(() => createContainer());
  const [queryClient] = useState(() => makeQueryClient());
  return (
    <WiringProvider container={container}>
      <QueryClientProvider client={queryClient}>
        <AppRoot>{children}</AppRoot>
      </QueryClientProvider>
    </WiringProvider>
  );
}
```

Create the container and the `QueryClient` inside the component, so the server never shares them between requests. `makeQueryClient` lives in `src/query/query-client.ts`, and `createContainer` is described in `stack/wiring.md`.

## App root and the set-once flag

```ts
// initial-boot-complete.state.ts
import { create } from "zustand";

const useBootCompleteStore = create(() => ({ complete: false }));

export const useInitialBootComplete = () => useBootCompleteStore((s) => s.complete);
export const markBootComplete = () => useBootCompleteStore.setState({ complete: true });
```

The store is private, and nothing exports a way to reset the flag. `markBootComplete` runs only in the browser, after boot succeeds, so the server never sets it.

The flag is module-level, so it outlives the container. That is correct only because `Providers` is mounted once, by the root layout, which never remounts. Never mount `Providers` anywhere else.

```tsx
// app-root.tsx
"use client";

export function AppRoot({ children }: { children: React.ReactNode }) {
  const coordinators = useCoordinators();
  useEffect(() => {
    coordinators.forEach((coordinator) => coordinator.start());
    return () => coordinators.forEach((coordinator) => coordinator.stop());
  }, [coordinators]);

  const bootComplete = useInitialBootComplete();
  return bootComplete ? <AppShell>{children}</AppShell> : <BootShell />;
}
```

`AppRoot` is the only View that may touch a Service, and only to keep coordinators alive.

## Boot Use Case

```ts
// src/use-cases/system/system-init.use-case.ts
export class SystemInitUseCase {
  private inflight: Promise<void> | null = null;
  private state: BootState = { status: "idle" };
  private listeners = new Set<() => void>();

  constructor(private readonly phases: BootPhases, private readonly services: Startable[]) {}

  getSnapshot = (): BootState => this.state;
  subscribe = (listener: () => void) => {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  };

  execute(): Promise<void> {
    return (this.inflight ??= this.run().finally(() => (this.inflight = null)));
  }

  private async run(): Promise<void> {
    this.set({ status: "running" });
    try {
      await this.phase("storage", () => this.phases.storage.execute());
      await this.phase("session", () => this.phases.session.execute());
      await this.phase("settings", () => this.phases.settings.execute());
      this.services.forEach((service) => service.stop());
      this.services.forEach((service) => service.start());
      this.set({ status: "done" });
    } catch (error) {
      this.set({ status: "failed", phase: error instanceof BootPhaseError ? error.phase : "unknown" });
      throw error;
    }
  }
}
```

- Creating the Use Case does not run boot; only `execute()` does.
- `inflight` makes overlapping calls, such as React Strict Mode's double effect, share one run.
- `phase(name, work)`, a private method left out above, wraps each step and rethrows a `BootPhaseError` from Contracts, so the shell knows which phase failed. A failed phase stops boot.
- Phase Use Cases fall back to cached local data on transient network errors when it exists.
- Boot phases read each dependency once. Anything that runs after boot subscribes.

## Triggering boot

```ts
// boot-shell.view-model.ts
export function useBootShellViewModel() {
  const { systemInit } = useUseCases();
  const state = useSyncExternalStore(systemInit.subscribe, systemInit.getSnapshot, systemInit.getSnapshot);

  const start = useCallback(async () => {
    try {
      await systemInit.execute();
      markBootComplete();
    } catch {
      // The failed state is already in the snapshot.
    }
  }, [systemInit]);

  return { start, isFailed: state.status === "failed", errorKey: bootErrorKey(state) };
}
```

| Entry point | Call |
|---|---|
| Cold start | `BootShell` calls `start()` in a mount effect |
| After sign-in or sign-out | `useAppShellViewModel`, which stays mounted. When the session's user id changes, it calls `queryClient.clear()` and then `systemInit.execute()` |
| Boot failure retry | The retry button calls `start()` again |

## Phase 2

- `AppShell` mounts `useDataChangeInvalidation()` once, renders the routed `children`, and hosts global overlays.
- Each overlay in `phase-2/overlays/` is presentation-only and reads its own ViewModel.
- On failure, the shells show translated copy from `errorKey`, never raw error text.
