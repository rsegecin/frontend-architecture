---
paths:
  - "**/*_notifier.dart"
  - "**/*_state.dart"
  - "**/*_controller.dart"
---

# Flutter: ViewModels

A ViewModel is a Riverpod notifier colocated with the widget it serves. It only talks to Use Cases.

## Naming

Mirror the widget file. Keep notifier, state, and controller files in the same directory as that widget.

| Widget | Notifier file | Notifier class | State file, when needed | Controller file, when needed |
|---|---|---|---|---|
| `order_form_screen.dart` | `order_form_screen_notifier.dart` | `OrderFormScreenNotifier` | `order_form_screen_state.dart` | `order_form_screen_controller.dart` |
| `depth_input_dialog.dart` | `depth_input_dialog_notifier.dart` | `DepthInputDialogNotifier` | `depth_input_dialog_state.dart` | |
| `sorting_filter.dart` | `sorting_filter_notifier.dart` | `SortingFilterNotifier` | | |

A screen notifier keeps the `_screen_` infix: `order_form_screen_notifier.dart`, never `order_form_notifier.dart`.

## Shape

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'settings_screen_notifier.g.dart';

@Riverpod(dependencies: [ExportOrdersUseCase])
class SettingsScreenNotifier extends _$SettingsScreenNotifier {
  @override
  void build() {
    ref.watch(exportOrdersUseCaseProvider);
  }

  bool get isExportAvailable =>
      ref.read(exportOrdersUseCaseProvider.notifier).canExport;

  Future<SettingsExportResult> exportOrders() async {
    if (!ref.mounted) {
      return SettingsExportResult.error;
    }

    final useCase = ref.read(exportOrdersUseCaseProvider.notifier);
    if (!useCase.canExport) {
      return SettingsExportResult.notAvailable;
    }

    try {
      await useCase.execute();
      return ref.mounted
          ? SettingsExportResult.success
          : SettingsExportResult.error;
    } catch (error, stackTrace) {
      log('[SettingsScreenNotifier] Export failed: $error\n$stackTrace');
      return SettingsExportResult.error;
    }
  }
}
```

When the notifier exposes data, return it from `build()`:

```dart
@Riverpod(dependencies: [UserSettingsUseCase])
class HomeScreenNotifier extends _$HomeScreenNotifier {
  @override
  HomeSettingsDTO build() {
    ref.watch(userSettingsUseCaseProvider);
    return ref.read(userSettingsUseCaseProvider.notifier).getHomeSettings();
  }
}
```

## Rules

- Use code generation for every notifier, and include its `part` file.
- Extend the generated base class, `_$ClassName`.
- In `build()`, `ref.watch` the Use Case providers the state depends on.
- Declare every Use Case the notifier reads in `@Riverpod(dependencies: [...])`, by class name.
- Action methods call a Use Case method with `ref.read(...notifier)`.
- Check `ref.mounted` after an async gap before touching `state` or `ref`.
- Result enums and view-state classes live in the `*_state.dart` file.
