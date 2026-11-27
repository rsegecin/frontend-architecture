---
paths:
  - "**/main.dart"
  - "**/app_root.dart"
  - "**/boot/**"
  - "**/*_init_usecase.dart"
---

# Flutter: App Boot

`core/cross-cutting.md` defines the boot rules. This file gives their Flutter form.

## Two-phase shell

| Phase | Widget | Shown |
|---|---|---|
| 1 | `BootShell` | Until `InitialBootComplete` is true |
| 2 | `AppShell` | After the first successful boot. Never unmounts on sign-in or sign-out. |

```text
lib/widgets/app/
  app_root.dart
  boot/
    initial_boot_complete_provider.dart
    phase_1/
      boot_shell.dart
      boot_shell_notifier.dart
    phase_2/
      app_shell.dart
      app_shell_notifier.dart
      overlays/
```

```dart
class AppRoot extends ConsumerWidget {
  const AppRoot({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    ref.watch(signOutCoordinatorProvider); // keeps the coordinator alive
    final bootComplete = ref.watch(initialBootCompleteProvider);
    return bootComplete ? const AppShell() : const BootShell();
  }
}
```

```dart
@Riverpod(keepAlive: true)
class InitialBootComplete extends _$InitialBootComplete {
  @override
  bool build() => false;

  void markComplete() => state = true;
}
```

## Boot Use Case

```dart
@Riverpod(keepAlive: true, dependencies: [/* every provider read in _init */])
class SystemInitUseCase extends _$SystemInitUseCase {
  Future<SystemInitState>? _inflight;

  @override
  AsyncValue<SystemInitState> build() => const AsyncValue.loading();

  Future<SystemInitState> execute() {
    return _inflight ??= _execute().whenComplete(() => _inflight = null);
  }

  Future<SystemInitState> _execute() async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(_init);
    return state.requireValue;
  }
}
```

- `build()` does not run boot.
- `_inflight` makes overlapping calls share one run.
- Inside `_init()`, use `await ref.read(...Provider.future)`, never `ref.watch`.
- Wrap each phase, so a failure carries the phase that failed. The shell maps it to localized copy.
- When a phase's provider is in `AsyncError`, call `ref.invalidate` on it before reading it again. Otherwise a retry re-awaits the same error.

## Triggering boot

Shells call a method on their notifier, which calls `execute()`.

```dart
@Riverpod(dependencies: [SystemInitUseCase])
class BootShellNotifier extends _$BootShellNotifier {
  @override
  AsyncValue<SystemInitState> build() => ref.watch(systemInitUseCaseProvider);

  Future<void> start() =>
      ref.read(systemInitUseCaseProvider.notifier).execute();
}
```

| Entry point | Call |
|---|---|
| Cold start | `BootShell` calls `start()` from `addPostFrameCallback` |
| After sign-in or sign-out | `AppShellNotifier`, which stays mounted |
| Boot failure retry | `start()` again. Do not invalidate the boot provider. |

## Phase 2 layout

Global overlay layers live in `AppShell`'s `builder` only. Each one watches its own notifier.

```dart
MaterialApp.router(
  builder: (context, child) => Stack(
    children: [
      child!,
      const AppBootOverlayLayer(),
    ],
  ),
);
```

## Boot failures

Show failure dialogs with a context below `MaterialApp`, through a navigator key. The `BootShell` build context sits above it.
