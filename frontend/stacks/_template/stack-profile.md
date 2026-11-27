# Stack Profile: {{stack name}}

This profile is blank. `setup-once.md` fills it in during the first session.

{{One line: framework, language, state library, and test tool, with major versions.}}

## Role map

Paths in the last column are relative to `.claude/rules/frontend-architecture/`.

| Core role | Concrete form | Location | Read first |
|---|---|---|---|
| View | {{component type}} | {{directories}} | {{`stack/views.md`}} |
| ViewModel | {{state holder, one per View}} | Next to its View | {{`stack/view-models.md`}} |
| Local UI state | {{mechanism}} | In or next to its View | {{`stack/views.md`}} |
| Use Case | {{form}} | {{directory and file suffix}} | {{`stack/use-cases.md`}} |
| Repository | {{form}} | {{remote and local directories}} | {{`stack/repositories.md`}} |
| Data Source | {{form, plus interface}} | {{directory}} | {{`stack/data-sources.md`}} |
| Service | {{form}} | {{directory and file suffix}} | {{`stack/services.md`}} |
| Util | {{form}} | {{directory}} | `core/logic-placement.md` |
| Contracts | {{form}} | {{directory}} | {{`stack/data-models.md`}} |

## Naming

{{How a ViewModel's file and name mirror its View, for screens, dialogs, and other components.}}

## Directory map

```text
{{source root and top-level directories, one per role}}
```

Create a directory only when its first file is added.

## Commands

| Task | Command |
|---|---|
| Generate code | {{command, or none}} |
| Analyze | {{command}} |
| Test | {{command}} |
