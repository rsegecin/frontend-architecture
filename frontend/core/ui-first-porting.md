# Architecture: UI-First Screen Porting

When the user asks to **pull**, **migrate**, **port**, or **scaffold** a screen from another source, build a layout shell, not a wired feature.

This rule applies only to that kind of request. A request to add, build, or create a screen is a normal feature, so build every layer it needs, as `layers.md` describes.

## Deliver

- The screen under the screens directory, in a feature folder that follows the user journey.
- Static structure, theming, spacing, typography, and navigation targets, with placeholders where behavior will attach later.
- Sample data or empty states, so the screen compiles and renders.
- Local UI state only if layout requires it, kept inside the View for this phase.
- Short `TODO` comments at the integration points where a ViewModel or Use Case will connect.

## Defer to a later wiring phase

- ViewModels, or any subscription to app state.
- Use Cases, Repositories, Services, and API or local persistence.
- Business logic and production data loading.
