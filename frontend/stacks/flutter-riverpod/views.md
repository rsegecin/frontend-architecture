---
paths:
  - "**/*_screen.dart"
  - "**/*_dialog.dart"
  - "**/screens/**"
  - "**/widgets/**"
---

# Flutter: Views

## Choosing a widget type

| Widget needs | Use |
|---|---|
| No app state, no local state | `StatelessWidget` |
| A ViewModel | `ConsumerWidget` |
| A ViewModel and local UI state | `HookConsumerWidget` |
| Local UI state only | `HookWidget` |

Reach for `StatefulWidget` or `ConsumerStatefulWidget` only when hooks cannot express the lifecycle.

## Reading a ViewModel

```dart
class SettingsScreen extends HookConsumerWidget {
  const SettingsScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    ref.watch(settingsScreenProvider);

    return SettingsMenuItem(
      onTap: () => ref.read(settingsScreenProvider.notifier).exportOrders(),
    );
  }
}
```

Use `ref.watch` to render state and `ref.read(...notifier)` inside callbacks. A widget never references a `*UseCaseProvider`, `*RepositoryProvider`, or `*ServiceProvider`.

## Local UI state

Default to hooks: `useTextEditingController`, `useScrollController`, `useAnimationController`, `useState`. Move a controller into a colocated `*_controller.dart` only when it carries logic of its own. Custom hooks live in `lib/hooks/`.

## Ownership paths

Shared UI lives in `lib/widgets/`:

```text
lib/widgets/
  app/      # global shell entry and shared MaterialApp config
    boot/   # two-phase boot
  dialogs/  # shared dialog building blocks
  inputs/
  layout/
```

Screen-group UI lives under `lib/screens/<group>/widgets/`:

```text
lib/screens/authentication/widgets/
lib/screens/dashboard/widgets/
  shell/    # subfolders once a real cluster exists
```

Other locations:

| Kind | Location |
|---|---|
| Asset path enums | `lib/assets/` |
| `TextInputFormatter` | `lib/utils/formatters/` |

```text
Bad:  lib/widgets/authentication/pin_input_widget.dart   # domain widget in shared UI
Good: lib/screens/authentication/widgets/pin_input_widget.dart

Good: lib/widgets/layout/app_scaffold.dart               # generic, even with one user today
```
