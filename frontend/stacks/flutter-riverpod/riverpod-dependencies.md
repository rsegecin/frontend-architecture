---
paths:
  - "**/*.dart"
---

# Flutter: Riverpod Dependencies and Reactivity

## 1. Using a provider obliges you to declare it, when scoped

If provider **A** reads provider **B** through `ref.read`, `ref.watch`, `ref.listen`, `ref.invalidate`, or `ref.exists`, then **B** must be accounted for in A's dependency contract:

| Provider B | What A must do |
|---|---|
| **Scoped**, meaning it has `@Riverpod(dependencies: [...])` | Add B's class or function name to A's `dependencies` |
| **Non-scoped**, meaning a plain `@riverpod` | Do not list it. Reads are allowed without listing. |

List the class name, such as `SyncOrdersUseCase`, not the generated provider variable.

If A reads any scoped provider, A must declare its own `dependencies`.

Merely using a provider counts. Even one `ref.read` in a side-effect method, `execute()`, or helper requires the declaration when B is scoped. Riverpod enforces this at runtime. `riverpod_lint` is optional and reports the same problem at analysis time.

## 2. Ref scope: reads in helpers count for the owner

A `Ref` belongs to the provider that created it. Any read through that `Ref`, including in interceptors, closures, private methods, or async callbacks, is attributed to the owning provider.

```dart
// HttpClient owns this ref. Reads inside the interceptor count for HttpClient.
instance.interceptors.add(AuthInterceptor(ref));
```

Moving `ref.read` into a class without `@Riverpod` does not avoid the requirement.

## 3. `dependencies` does not mean "listen to everything"

Listing a provider in `dependencies` does not subscribe to its changes. It only declares the override and scoping graph. Reactivity comes only from how you read:

| API | Reacts when B changes? |
|---|---|
| `ref.watch(bProvider)` | Yes: A rebuilds |
| `ref.listen(bProvider, ...)` | Yes: the callback fires, with no rebuild |
| `ref.read(bProvider)` | No: one-shot read |
| `ref.read(bProvider.notifier).someMethod()` | No, unless A also watches B elsewhere |
| B listed in `dependencies` only | No: declaration alone never triggers rebuilds |

```dart
@Riverpod(dependencies: [UserSettingsUseCase])
class HomeScreenNotifier extends _$HomeScreenNotifier {
  @override
  HomeSettingsDTO build() {
    ref.watch(userSettingsUseCaseProvider); // reactive
    return ref.read(userSettingsUseCaseProvider.notifier).getHomeSettings();
  }

  Future<void> onSave(HomeSettingsDTO settings) async {
    // Side effect. A read is correct here.
    await ref.read(userSettingsUseCaseProvider.notifier).save(settings);
  }
}
```

Common mistakes:

- Adding a provider to `dependencies` and expecting state to update. Add `ref.watch` in `build()`.
- Using `ref.watch` in `execute()` or action methods. Use `ref.read` for one-shot work.

## 4. Breaking dependency cycles

If declaring every used provider creates a cycle, restructure. Do not omit dependencies to silence the error.

To let a lower layer request higher-level work, use a signal and a coordinator:

```dart
// lib/data/signals/sign_out_signal.dart: non-scoped, so anything may raise it.
@Riverpod(keepAlive: true)
class SignOutSignal extends _$SignOutSignal {
  @override
  int build() => 0;

  void request() => state++;
}
```

```dart
// lib/services/session/sign_out_coordinator.dart: outside the cycle.
@Riverpod(keepAlive: true, dependencies: [SignOutUseCase])
class SignOutCoordinator extends _$SignOutCoordinator {
  @override
  void build() {
    ref.listen(signOutSignalProvider, (previous, next) {
      ref.read(signOutUseCaseProvider.notifier).execute();
    });
  }
}
```

The root widget must `ref.watch` the coordinator, so it stays alive.

## Quick checklist

1. Every scoped provider read is listed in `dependencies`, by class name.
2. Reads through a passed `Ref` are listed on the Ref's owner, not the helper.
3. State that must react uses `ref.watch` in `build()`.
4. One-shot actions and `execute()` use `ref.read`.
