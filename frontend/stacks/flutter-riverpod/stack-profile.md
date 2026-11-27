# Stack Profile: Flutter + Riverpod

Flutter with Riverpod 3 code generation: `hooks_riverpod`, `flutter_hooks`, `riverpod_annotation`, `riverpod_generator`, `build_runner`, and `json_serializable` for DTOs.

## Role map

Paths in the last column are relative to `.claude/rules/frontend-architecture/`.

| Core role | Concrete form | Location | Read first |
|---|---|---|---|
| View | Flutter widget | `lib/screens/<group>/`, `lib/widgets/<role>/` | `stack/views.md` |
| ViewModel | `@Riverpod` class named `*Notifier` | Next to its View | `stack/view-models.md` |
| Local UI state | Hooks, then `*_controller.dart` | In or next to its View | `stack/views.md` |
| Use Case | `@Riverpod` class named `*UseCase` | `lib/usecases/<domain>/*_usecase.dart` | `stack/use-cases.md` |
| Repository | `@Riverpod` class named `*Repository` | `lib/repositories/api/`, `lib/repositories/local/<entity>/` | `stack/repositories.md` |
| Data Source | `keepAlive` `@Riverpod` class plus interface | `lib/datasources/<capability>/` | `stack/data-sources.md` |
| Service | `keepAlive` `@Riverpod` class named `*Service` | `lib/services/<capability>/*_service.dart` | `stack/services.md` |
| Util | Static class or extension, no provider | `lib/utils/<area>/` | `core/logic-placement.md` |
| Contracts | Plain Dart classes and enums | `lib/data/` | `stack/data-models.md` |

Every `@Riverpod` class also follows `stack/riverpod-dependencies.md`. App startup follows `stack/app-boot.md`, and tests follow `stack/tests.md`.

## Directory map

```text
lib/
  main.dart
  assets/        # asset path enums
  config/        # constants, endpoints
  data/          # Contracts
  datasources/
  hooks/         # custom hooks
  i18n/
  repositories/
  routing/
  screens/
  services/
  theme/
  usecases/
  utils/
  widgets/       # shared UI
test/            # mirrors lib/
```

Create a directory only when its first file is added.

## Commands

| Task | Command |
|---|---|
| Generate code | `dart run build_runner build --delete-conflicting-outputs` |
| Analyze | `flutter analyze` |
| Test | `flutter test` |

Run the generate command after creating or changing anything annotated with `@Riverpod` or `@JsonSerializable`. Never edit `*.g.dart` files by hand.

## Provider names

The generator drops a trailing `Notifier` from the class name, and keeps every other suffix.

| Class | Generated provider |
|---|---|
| `SettingsScreenNotifier` | `settingsScreenProvider` |
| `ExportOrdersUseCase` | `exportOrdersUseCaseProvider` |
| `ApiOrdersRepository` | `apiOrdersRepositoryProvider` |
