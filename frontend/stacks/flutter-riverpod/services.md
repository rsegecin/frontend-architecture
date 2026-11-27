---
paths:
  - "**/services/**"
  - "**/*_service.dart"
---

# Flutter: Services

A Service is a `keepAlive` `@Riverpod` class in `lib/services/<capability>/*_service.dart`. Add a sibling `*_service_int.dart` when other code depends on its surface.

## Shape

```dart
part 'orders_sync_service.g.dart';

@Riverpod(keepAlive: true, dependencies: [SyncOrdersUseCase])
class OrdersSyncService extends _$OrdersSyncService {
  Timer? _timer;

  @override
  void build() {
    ref.watch(syncOrdersUseCaseProvider);

    _timer = Timer.periodic(const Duration(minutes: 5), (_) => _sync());
    ref.onDispose(() => _timer?.cancel());
  }

  Future<void> _sync() async {
    try {
      await ref.read(syncOrdersUseCaseProvider.notifier).execute();
    } catch (error, stackTrace) {
      log('[OrdersSyncService] Sync failed: $error\n$stackTrace');
    }
  }
}
```

## Rules

- Always `keepAlive: true`.
- Release every timer and subscription in `ref.onDispose`.
- Work goes through `ref.read(...UseCaseProvider.notifier)`. Never reference a Repository provider.
- A Service may subscribe directly to the external event that triggers it, such as a connectivity stream. What it does in response goes through a Use Case.
- The boot Use Case starts and stops Services. Widgets and notifiers do not.
- To surface a Service's state in the UI, expose it from a Use Case and watch that.
