---
paths:
  - "**/*_test.dart"
  - "**/test/helpers/**"
---

# Flutter: Tests

- `test/` mirrors `lib/`. A test for `lib/usecases/system/x_usecase.dart` goes in `test/usecases/system/x_usecase_test.dart`.
- Shared fakes live in `test/helpers/`, named `fake_*.dart`.
- Fake at the Data Source or Repository boundary, and run the real Use Case against it.

## Fakes

Subclass the provider and stub `build()`, so no real SDK runs.

```dart
class FakeLocalOrdersRepository extends LocalOrdersRepository {
  FakeLocalOrdersRepository({this.error});

  final Object? error;

  @override
  void build() {}

  @override
  Future<List<OrderDTO>> getByUserId({required String userId}) async {
    final error = this.error;
    if (error != null) throw error;
    return const [];
  }
}
```

## Containers

```dart
container = ProviderContainer.test(
  retry: (_, _) => null,
  overrides: [
    localOrdersRepositoryProvider.overrideWith(() => fakeOrders),
  ],
);

await container.read(exportOrdersUseCaseProvider.notifier).execute();
```

`retry: (_, _) => null` turns off automatic retry, so a failing provider settles in `AsyncError` right away.
