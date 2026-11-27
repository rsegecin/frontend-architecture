# Stack Profile: {{stack name}}

This profile is blank. `setup-once.md` fills it in during the first session.

{{One line: runtime, HTTP framework, validation library, database access, and test tool, with major versions.}}

## Role map

Paths in the last column are relative to `.claude/rules/backend-architecture/`.

| Core role | Concrete form | Location | Read first |
|---|---|---|---|
| Route | {{route or controller form}} | {{directory and file suffix}} | {{`stack/controllers.md`}} |
| Contract | {{schema library and naming}} | {{directory}} | {{`stack/controllers.md`}} |
| Application Service | {{form}} | {{directory and file suffix}} | {{`stack/services.md`}} |
| Domain | {{rich entities, or types and pure functions}} | {{directory}} | {{`stack/domain.md`}} |
| Repository | {{form, plus interface}} | {{directory and file suffix}} | {{`stack/repositories.md`}} |
| Integration | {{interface and implementations}} | {{directory}} | {{`stack/modules.md`}} |
| System entry | {{cron, queue worker, event handler, or none yet}} | {{directory}} | {{`stack/modules.md`}} |
| Module | {{composition root form}} | {{file}} | {{`stack/modules.md`}} |
| Infrastructure | {{how app-wide instances are created and exposed}} | {{directory}} | {{`stack/modules.md`}} |
| Util | {{form}} | {{directory}} | `core/logic-placement.md` |
| Shared Contracts | {{error classes, error codes, shared enums}} | {{directory}} | `core/contracts-and-errors.md` |

## Errors and pagination

- Error body: {{shape}}
- Validation status: {{400 or 422}}
- Who translates database conflicts: {{Repository or error handler}}
- Pagination: {{request params and response shape}}

## Naming

{{File and export names for each role, and the table and column naming convention.}}

## Directory map

```text
{{source root and top-level directories, one per role}}
```

Create a directory only when its first file is added.

## Commands

| Task | Command |
|---|---|
| Generate OpenAPI | {{command}} |
| Database migration | {{command}} |
| Generate database client or types | {{command, or none}} |
| Analyze | {{lint and typecheck commands}} |
| Test | {{command}} |
