---
paths:
  - "**/*_repository.dart"
  - "**/repositories/**"
---

# Flutter: Repositories

Repositories are `@Riverpod` classes that delegate to Data Sources.

## Layout

```text
lib/repositories/
  api/
    api_orders_repository.dart
  local/
    orders/
      local_orders_repository.dart
      local_orders_repository_int.dart
```

| Kind | File | Interface |
|---|---|---|
| Remote | `api/api_<entity>_repository.dart` | None |
| Local | `local/<entity>/local_<entity>_repository.dart` | `local_<entity>_repository_int.dart`, class `Local<Entity>RepositoryInt` |

## Remote shape

```dart
part 'api_orders_repository.g.dart';

@Riverpod(dependencies: [HttpClient])
class ApiOrdersRepository extends _$ApiOrdersRepository {
  HttpClientInt get httpClient => ref.read(httpClientProvider.notifier);

  @override
  void build() {
    ref.watch(httpClientProvider);
  }

  Future<OrderResponseDTO> getOrder({required String id}) async {
    final json = await httpClient.get(Endpoints.order(id));
    return OrderResponseDTO.fromJson(json);
  }
}
```

## Local shape

```dart
part 'local_orders_repository.g.dart';

@Riverpod(dependencies: [LocalDB])
class LocalOrdersRepository extends _$LocalOrdersRepository
    implements LocalOrdersRepositoryInt {
  LocalDBInt get datasource => ref.read(localDBProvider.notifier);

  @override
  void build() {
    ref.watch(localDBProvider);
  }

  @override
  Future<List<OrderDTO>> getByUserId({required String userId}) async {
    final records = await datasource.query(
      table: OrderTable.kTableName,
      conditions: {OrderTable.kUserId: userId},
    );
    return records.map(OrderTable.fromMap).map(_mapper.toDto).toList();
  }
}
```

## Rules

- Hold the Data Source in a getter typed to its interface, as `datasource` does above.
- `build()` watches the Data Source. Methods use the getter.
- Return Contracts. Parse JSON and map rows here, never above this layer.
- Add `keepAlive: true` only when the Repository caches state that must survive between readers.
