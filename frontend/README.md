# Frontend Architecture Rules

Claude Code rules that give a front-end project one layered architecture:

```text
View -> ViewModel -> Use Case -> Repository -> Data Source
Service -> Use Case -> Repository -> Data Source
```

Copy them into a project, and Claude Code follows them in every session. The core rules work on any stack. Flutter with Riverpod has a ready-made profile, and React with Next.js has three: one built on Jotai, one built on Zustand, and a lighter one built on domain hooks. On any other stack, Claude writes the profile itself in the first session.

## Adopt it

Run these from the root of the target project, with `KIT` pointing at this repository.

**Flutter with Riverpod:**

```sh
KIT=/path/to/frontend-architecture
RULES=.claude/rules/frontend-architecture
mkdir -p "$RULES/core" "$RULES/stack"
cp -R "$KIT/frontend/core/." "$RULES/core/"
cp -R "$KIT/frontend/stacks/flutter-riverpod/." "$RULES/stack/"
cp "$KIT/frontend/setup/"*.md "$RULES/"
```

**React + Next.js** (App Router, TanStack Query, Jotai, zod):

```sh
KIT=/path/to/frontend-architecture
RULES=.claude/rules/frontend-architecture
mkdir -p "$RULES/core" "$RULES/stack"
cp -R "$KIT/frontend/core/." "$RULES/core/"
cp -R "$KIT/frontend/stacks/react/." "$RULES/stack/"
cp "$KIT/frontend/setup/"*.md "$RULES/"
```

**React + Next.js with Zustand** (App Router, TanStack Query, Zustand, zod):

```sh
KIT=/path/to/frontend-architecture
RULES=.claude/rules/frontend-architecture
mkdir -p "$RULES/core" "$RULES/stack"
cp -R "$KIT/frontend/core/." "$RULES/core/"
cp -R "$KIT/frontend/stacks/react-zustand/." "$RULES/stack/"
cp "$KIT/frontend/setup/"*.md "$RULES/"
```

**Next.js with domain hooks** (App Router, TanStack Query hooks and static services per domain module over axios, Zustand, next-intl, zod):

```sh
KIT=/path/to/frontend-architecture
RULES=.claude/rules/frontend-architecture
mkdir -p "$RULES/core" "$RULES/stack"
cp -R "$KIT/frontend/core/." "$RULES/core/"
cp -R "$KIT/frontend/stacks/nextjs-domain-hooks/." "$RULES/stack/"
cp "$KIT/frontend/setup/"*.md "$RULES/"
```

**Any other stack:** run the same commands, with `stacks/_template` in place of `stacks/flutter-riverpod`.

Commit the copied files so your team gets them. No change to your `CLAUDE.md` is needed.

Copy the files rather than symlinking them. A symlink pointing outside the project needs an approval, and after that Claude Code still skips every rule scoped with `paths`.

## What happens in the first session

Before its first file change, Claude runs `setup-once.md`:

1. It inspects the project, and fills in the stack profile if you used the template.
2. It fills in `project-context.md`: platforms, domain glossary, API contract source, related projects, and logger names. Anything it can't find stays marked `unknown, ask when relevant`.
3. It lists existing code that breaks the rules.
4. It replaces `setup-once.md` with a one-line note, so setup doesn't run again.

Claude Code protects `.claude/`, so you get a permission prompt for these writes, and `permissions.allow` rules can't approve them in advance. Choose the option that allows edits to the `.claude` folder for the session. If you refuse, Claude still finishes your task, then prints the files for you to save. Setup runs again next session until they're saved.

Run `/context` to see which rule files are loaded.

## What's in here

| Path | Copied to | Purpose |
|---|---|---|
| `core/` | `$RULES/core/` | Stack-neutral rules. Always loaded. |
| `setup/` | `$RULES/` | Setup procedure and project facts. Copied once. |
| `stacks/flutter-riverpod/` | `$RULES/stack/` | Role map, plus rules that load when Claude touches matching files. |
| `stacks/react/` | `$RULES/stack/` | React + Next.js role map, matching rules, and a packages-by-layer guide. |
| `stacks/react-zustand/` | `$RULES/stack/` | The same React + Next.js profile, with a plain container for wiring and Zustand for view state. |
| `stacks/nextjs-domain-hooks/` | `$RULES/stack/` | A lighter Next.js profile: domain hooks play the Use Case, static services registered in each domain module replace the container, and next-intl handles locales. |
| `stacks/_template/` | `$RULES/stack/` | Blank profile for other stacks. |

This README isn't copied, because every `.md` file under `.claude/rules/` gets loaded.

## Customize

- Put project facts in `project-context.md`.
- Put stack conventions in `stack/`.
- Leave `core/` unchanged where you can, so upgrades stay a plain copy.

## Upgrade

Re-copy `core/` only:

```sh
cp -R "$KIT/frontend/core/." "$RULES/core/"
```

`stack/` and `project-context.md` hold your project's content, so merge any changes to those by hand.

## Limits

Claude Code treats rules as context, not enforcement, so Claude can still miss one. Path-scoped rules also load only after a matching file has been touched. To cover this, `core/layers.md` tells Claude to Read a role's file before creating the first file in that role. For hard guarantees, add lint rules or CI checks.
