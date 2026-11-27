---
paths:
  - "**/*_usecase.dart"
  - "**/usecases/**"
---

# Flutter: Use Cases

A Use Case is a `@Riverpod` class named `*UseCase`, in `lib/usecases/<domain>/*_usecase.dart`.

## Shape

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'export_orders_usecase.g.dart';

@Riverpod(dependencies: [LocalOrdersRepository, CurrentUserRepository])
class ExportOrdersUseCase extends _$ExportOrdersUseCase {
  @override
  void build() {
    ref.watch(localOrdersRepositoryProvider);
    ref.watch(currentUserRepositoryProvider);
  }

  bool get canExport {
    final userId = ref.read(currentUserRepositoryProvider).value?.id;
    return userId != null && userId.isNotEmpty;
  }

  Future<void> execute() async {
    if (!canExport) {
      throw StateError('User is not initialized');
    }

    log('[ExportOrdersUseCase] Starting export');

    final orders = await ref
        .read(localOrdersRepositoryProvider.notifier)
        .getAll();
    await OrdersExporter.share(orders);
  }
}
```

The caller goes through `.notifier`, because the methods live on the class:

```dart
await ref.read(exportOrdersUseCaseProvider.notifier).execute();
```

## Rules

- Never write a Use Case as a plain class. Every Use Case is a generated provider, so its dependencies stay declared and it can be overridden in tests.
- `build()` watches what the Use Case depends on. Use `ref.read` inside `execute()` and other methods.
- Make the Use Case's state its `build()` return type when consumers need to react to it. Use `void` when it only exposes methods.
- Expose stream-driven state from a Use Case, so ViewModels can watch it without touching a Service.
- Return Contracts, and keep view-specific strings out.

## keepAlive

Default to `@Riverpod(dependencies: [...])`. Add `keepAlive: true` when the Use Case:

- Caches state that consumers share.
- Owns a stream or subscription.
- Must finish its work after its caller is gone, such as signing out.

In a `keepAlive` Use Case, check `ref.mounted` after async gaps too.
