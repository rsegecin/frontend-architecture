# Architecture: Services, Use Cases, and Utils

Use this rule to pick the right layer for logic that is not UI.

## Quick decision

| Layer | Use when | Lifetime |
|---|---|---|
| **Service** | Something must stay alive and react over time: timers, listeners, background sync, device streams | As long as its work runs |
| **Use Case** | Business orchestration for a user or system action | On demand, unless it holds state or must outlive its caller |
| **Util** | Stateless technical work with no business rules | None: no instance, no injection |

## Services

- Services are side-effect triggers and external entry points. They are actors, not a business-logic layer.
- A Service calls Use Cases to do its work. It never modifies a Repository or keeps a reference to one, and it never holds UI state.
- Automated task orchestration belongs in Use Cases, so both ViewModels and Services can trigger the same logic.
- One-shot actions, file helpers, and static wrappers are not Services. If nothing needs to run in the background, it is not a Service.

## Use Cases

- They orchestrate business logic: validate preconditions, read Repositories and other Use Cases, and coordinate a feature action.
- They are the entry point for user flows (ViewModel -> Use Case) and system flows (Service -> Use Case).
- They own rules such as "can this be exported?", "the user must be signed in", and "which data to fetch".

## Utils

- Utils are pure technical helpers: formatting, file I/O, document generation, compression, platform share sheets.
- Utils have no injected dependencies, no Repository access, and no feature preconditions.
- They are called from Use Cases or Repositories. A View or ViewModel may not call one directly when the action involves app data or business rules.
- Name them with a descriptive noun, such as `Zipper` or `OrdersExporter`, never `*Service`.

## Anti-patterns

```text
Bad:  OrdersExportService with a static share() method
Why:  It is one-shot, so it is not a Service.
Good: ExportOrdersUseCase decides whether export is allowed; OrdersExporter does the file work.

Bad:  View -> OrdersExporter.share(...)
Why:  The action needs app data and a business check.
Good: View -> ViewModel -> ExportOrdersUseCase -> OrdersExporter
```

## Organizing Use Cases

Group Use Cases by feature domain or capability, not by Repository layer.

```text
usecases/
  authentication/     # feature journey, partitioned by sub-journey
    login/
    signup/
    password_reset/
    session/
  orders/             # large domain, partitioned by capability
    create/
    query/
    manage/
    sync/
  places/             # reusable capability, used by several screens
  settings/           # small domain, flat
  system/             # app bootstrap and cross-cutting init
```

- **Feature journey** with several related actions: a feature folder.
- **Reusable capability** used by several screens: a capability folder.
- **App bootstrap or cross-cutting init**: `system/`.
- **New Use Case in an existing domain**: that domain's folder.
- **New domain**: create the folder only when adding its first Use Case.
- **Crowded folder**: partition by user journey, capability, or pipeline phase, once a real cluster exists. Keep helpers that are not Use Cases in an `internal/` subfolder.

Avoid:

- Flat files at the `usecases/` root when a domain folder exists.
- `cross_feature/`, `shared/`, or `common/` buckets. Name the capability instead.
- Mirroring the nesting of the screens directory.
- `local/` versus `remote/` splits, which mirror Repositories, not capabilities.
- Folders for features that do not exist yet.
