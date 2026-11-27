---
paths:
  - "**/datasources/**"
---

# Flutter: Data Sources

A Data Source is a `keepAlive` `@Riverpod` class that wraps one SDK or platform API, behind an interface.

## Layout

```text
lib/datasources/
  gps/
    gps_datasource.dart
    gps_datasource_int.dart
  http_resolver/
    http_client.dart
    http_client_int.dart
    helpers/
  local_database/
```

Name the interface class `<Implementation>Int`, in a sibling `*_int.dart` file.

## Shape

```dart
// gps_datasource_int.dart
abstract class GpsDatasourceInt {
  Future<bool> hasLocationPermission();

  Stream<GpsReading> watchReadings({int distanceFilterMeters = 1});
}
```

```dart
// gps_datasource.dart
part 'gps_datasource.g.dart';

@Riverpod(keepAlive: true)
class GpsDatasource extends _$GpsDatasource implements GpsDatasourceInt {
  @override
  void build() {}

  @override
  Future<bool> hasLocationPermission() async {
    final permission = await Geolocator.checkPermission();
    return permission == LocationPermission.always ||
        permission == LocationPermission.whileInUse;
  }

  @override
  Stream<GpsReading> watchReadings({int distanceFilterMeters = 1}) {
    // Convert SDK types to Contracts here.
  }
}
```

## Rules

- Always `keepAlive: true`. A Data Source owns a connection, a handle, or an SDK session.
- Import SDK packages here and nowhere above this layer.
- Convert SDK types to Contracts before returning them.
- Catch SDK exceptions and throw a class from `lib/data/errors/`, so no caller imports this layer for error types.
- Read tokens through another Data Source, such as secure storage. Never import from `lib/usecases/`.
- When a failure must trigger higher-level work, raise a signal. See `riverpod-dependencies.md`.
