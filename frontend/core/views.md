# Architecture: Views and ViewModels

## One ViewModel per View

- Each screen, dialog, or self-contained component that reads app or user data has its own ViewModel.
- The ViewModel lives in the same directory as the View it serves, and its file name mirrors the View's file name. The ViewModel file listed in `stack/stack-profile.md` gives the exact pattern.
- Keep the View's state and controller files in that same directory.
- The ViewModel subscribes to the Use Cases it needs and exposes view-specific state and action methods.
- Shared business logic stays in Use Cases. A ViewModel only calls them and maps their results to view state, including errors.

## Local UI state

- Purely local state (text input, scroll position, animation, ephemeral layout state) stays in the View or a colocated controller.
- Never place UI-control objects or their lifecycle in a ViewModel.
- Controllers handle UI logic only, never business logic.

## UI ownership

Place UI files by who owns them, not by file type alone.

| Bucket | Holds | Grouped by |
|---|---|---|
| **Shared UI** | Generic primitives with no domain meaning, and UI reused across unrelated screen groups | Role: layout, inputs, dialogs |
| **Screen-group UI** | Components that only make sense inside one screen group | Inside that group's directory |

To place a file:

1. Is it UI, or the ViewModel of UI? If not, it goes with assets or Utils, not with UI. Asset references and input formatters are not UI.
2. Is it a generic primitive usable without domain context? It goes in shared UI, even if only one group uses it today.
3. Is it owned by one screen group? It goes in that group's directory.
4. When a screen-group component gets reused by an unrelated group, promote it to shared UI, in a folder named after its capability.

## Screens

- One screen per file.
- Group screen directories by user journey or feature, never one flat folder of unrelated screens.
- Keep the stack's screen suffix on every screen file, so file searches find them all.
