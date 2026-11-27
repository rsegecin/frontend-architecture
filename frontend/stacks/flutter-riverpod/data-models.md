---
paths:
  - "**/data/**"
---

# Flutter: Contracts

`lib/data/` holds the Contracts role. Files here import no other layer, and no SDK other than serialization annotations.

## Layout

```text
lib/data/
  errors/              # typed errors that cross layers
  signals/             # dependency-free request signals
  local/
    domain/            # app-level models
    enums/
    tables/            # local persistence records
  remote/
    api/<feature>/     # request and response DTOs, by feature
    models/            # DTOs shared across features
  mappers/
    to_domain/
    to_dtos/
    to_tables/
```

## DTOs

Files end in `_dto.dart`, and classes end in `DTO`.

```dart
part 'order_dto.g.dart';

@JsonSerializable()
class OrderDTO extends Equatable {
  const OrderDTO({this.id, this.total});

  @JsonKey()
  final String? id;
  @JsonKey()
  final num? total;

  factory OrderDTO.fromJson(Map<String, dynamic> json) =>
      _$OrderDTOFromJson(json);

  Map<String, dynamic> toJson() => _$OrderDTOToJson(this);

  @override
  List<Object?> get props => [id, total];
}
```

## Tables

Files end in `_table.dart`. A table class declares its names as `k*` constants, and provides `toMap` and `fromMap`.

```dart
class OrderTable {
  static const String kTableName = 'orders';
  static const String kId = 'id';
  static const String kUserId = 'userId';

  final String id;
  final String userId;

  const OrderTable({required this.id, required this.userId});

  factory OrderTable.fromMap(Map<String, dynamic> map) => OrderTable(
    id: map[kId] as String,
    userId: map[kUserId] as String,
  );

  Map<String, dynamic> toMap() => {kId: id, kUserId: userId};
}
```

## Mappers

A mapper is a `final class` with `const` constructor and pure methods. It lives in the `mappers/` folder named after its target form.

```dart
final class OrderTableMapper {
  const OrderTableMapper();

  OrderDTO toDto(OrderTable row) => OrderDTO(id: row.id);
}
```

## Errors

Put errors that cross layers in `lib/data/errors/`, extending a common base class. Never import an HTTP SDK here. The Data Source converts SDK failures into these types.
