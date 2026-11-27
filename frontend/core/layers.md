# Architecture: Layers

Every feature in this project follows this layered architecture. Role names here are stack-neutral. `stack/stack-profile.md` maps each role to this project's directories, file names, and code shape.

Paths in these rules are relative to `.claude/rules/frontend-architecture/`, unless they name a project source directory.

## Roles

| Role | Job |
|---|---|
| **View** | Renders state and forwards user input. Screens, dialogs, components. |
| **ViewModel** | Holds view-specific state and action methods for one View, and bridges it to Use Cases. |
| **Local UI state** | Ephemeral state only: input fields, scroll position, animation, layout. |
| **Use Case** | The only place for business orchestration. Entry point for user and system actions. |
| **Repository** | The data boundary. Exposes data operations and delegates to Data Sources. |
| **Data Source** | Wraps one external system: HTTP, local database, secure storage, device sensors. |
| **Service** | Long-lived system actor: timers, listeners, background sync, device streams. |
| **Util** | Stateless technical helper with no business rules. |
| **Contracts** | Models, enums, and typed errors that depend on nothing. |

## Flow

```text
User path:    View -> ViewModel -> Use Case -> Repository -> Data Source
System path:  Service -> Use Case -> Repository -> Data Source
```

Never skip a layer in either chain.

## Who may call whom

| Caller | May call | Must never call |
|---|---|---|
| View | Its ViewModel, local UI state | Use Case, Repository, Data Source, Service |
| ViewModel | Use Cases | Repository, Data Source, Service |
| Use Case | Repositories, other Use Cases, Utils | View, ViewModel, Data Source, Service |
| Service | Use Cases, Utils, the external events that trigger it | Repository, Data Source, View, ViewModel |
| Repository | Data Sources, Utils | Use Case, Service, View, ViewModel |
| Data Source | External systems, Utils | Every role above it |
| Util | Other Utils | Every other role |

- Contracts sit outside the chain. Any role may import them, because "never skip a layer" governs calls, not shared types.
- The only Use Case that may touch Services is the boot Use Case, which starts and stops them. See `core/cross-cutting.md`.
- The only View that may touch a Service is the app root, which keeps coordinators alive. See `core/cross-cutting.md`.
- To show Service-driven state, a ViewModel watches a Use Case that exposes it.

## What each layer must not contain

- **View**: business rules, data access, or calls to anything but its ViewModel.
- **ViewModel**: business logic. For a side effect, call a Use Case method. Mapping Use Case results and errors to view state is its job.
- **Use Case**: UI logic of any kind: component concerns, presentation formatting, view-specific strings, or navigation.
- **Service**: UI state, or direct references to Repositories.

Use Cases return Contracts, never raw API payloads. Keep each Use Case focused on one responsibility whenever possible.

## Reactivity

When a Use Case changes data, every ViewModel watching that chain must update automatically, whether a ViewModel or a Service triggered the change. Do not add manual refreshes or polling to compensate for a broken chain.

## Building a feature

Create layers in this order, skipping any that already exist:

1. Contracts
2. Data Source, only for a new external system
3. Repository
4. Use Case
5. ViewModel
6. View

Before creating a file in a role, Read the file that `stack/stack-profile.md` lists for that role. Path-scoped guidance loads only after a matching file has been touched, so it cannot shape the first file you write in a role.

After changing anything the stack generates code from, run the generate command in `stack/stack-profile.md`.

## Existing code that breaks these rules

Existing violations are not precedent. New code follows this file even when nearby code does not. Mention the violation to the user, and do not refactor it unless asked.
