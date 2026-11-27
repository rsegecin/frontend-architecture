# Architecture: Data and Contracts

## Repositories

- A Repository is the data boundary. Everything above it reads and writes through a Repository.
- It delegates network, storage, and device access to Data Sources, and holds no SDK code.
- It takes no business decisions. Preconditions and "which data" choices belong to Use Cases.
- Keep remote and local Repositories separate, one per entity.

## Data Sources

- A Data Source wraps one external system.
- It has an interface, so that Repositories depend on the interface and tests can replace it.
- It holds no business rules, and knows nothing about who calls it.
- Credentials for a transport come from another Data Source, never from a Use Case.
- It catches SDK and transport failures, and rethrows them as Contract errors. SDK error types never leave a Data Source.

## Contracts

| Kind | Purpose |
|---|---|
| Domain models | App-level types shared across layers |
| DTOs | Wire-shaped types for remote payloads |
| Persistence records | Rows or documents as stored locally |
| Mappers | Pure conversions between the above, grouped by target form |
| Enums | Shared closed sets of values |
| Typed errors | Failures that cross layers |

Contracts import nothing from any other role, and no SDK types. Any role may import them.

## Typed errors

- An error that crosses more than one layer lives in Contracts.
- An error handled only by the adjacent layer may sit beside its thrower, such as a Use Case's validation error.
- Data Sources translate transport failures into Contract errors.
- ViewModels map errors to view state. Views display that state, never raw error text.

## Local schema changes

Every schema version bump adds a migration that shipped installs replay forever. Add a new version only when the current one is actually in users' hands.

Before bumping the schema version:

1. Find the latest release and the schema version it shipped with:

```sh
git tag --sort=-creatordate | head
git show <latest-tag>:<path-to-schema-version-constant>
```

2. Decide:

| Current schema version | Action |
|---|---|
| Shipped in the latest release tag | Add a new version and its migration |
| Only exists in unreleased commits | Merge the change into that version's existing migration. Do not bump again. |

3. Fresh installs use the schema creation path, so keep it in sync with the migration either way.

After merging into an unreleased version, dev and test devices that ran interim builds may hold a partially migrated schema. Reinstall the app or clear its data on those devices. Production is unaffected, because that version never shipped.
