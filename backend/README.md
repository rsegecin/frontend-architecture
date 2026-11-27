# Backend Architecture Rules

Claude Code rules that give an HTTP API one layered architecture:

```text
Route -> Application Service -> Repository -> Database
                             -> Integration -> External system
System entry -> Application Service -> Repository / Integration
```

They work like the rules in `frontend/`: copy them into a project, and Claude Code follows them in every session. The core rules work on any backend stack. Fastify with zod has a ready-made profile. On any other stack, Claude writes the profile itself in the first session.

## Adopt it

Run these from the root of the target API, with `KIT` pointing at this repository.

**Fastify with bounded contexts** (zod Contracts, Kysely and a unit of work, rich domain entities, domain events, BullMQ jobs, Vitest):

```sh
KIT=/path/to/frontend-architecture
RULES=.claude/rules/backend-architecture
mkdir -p "$RULES/core" "$RULES/stack"
cp -R "$KIT/backend/core/." "$RULES/core/"
cp -R "$KIT/backend/stacks/fastify-contexts/." "$RULES/stack/"
cp "$KIT/backend/setup/"*.md "$RULES/"
```

**Any other stack:** run the same commands, with `backend/stacks/_template` in place of the stack folder.

In a monorepo, run them from the API's own folder, so the rules load only for the API.

Commit the copied files so your team gets them. Copy them rather than symlinking them, for the same reasons as the frontend rules.

## What happens in the first session

Before its first file change, Claude runs `setup-once.md`: it fills in the stack profile if you used the template, fills in `project-context.md` (runtime, consumers, authentication, database, related projects), lists existing code that breaks the rules, and retires `setup-once.md`. The permission prompt for writes under `.claude/` works as described in `frontend/README.md`.

## What's in here

| Path | Copied to | Purpose |
|---|---|---|
| `core/` | `$RULES/core/` | Stack-neutral rules: layers, where logic goes, contracts and errors, cross-cutting rules. Always loaded. |
| `setup/` | `$RULES/` | Setup procedure and project facts. Copied once. |
| `stacks/fastify-contexts/` | `$RULES/stack/` | Fastify profile: one folder per bounded context with `presentation/`, `application/`, `domain/`, `infra/`. |
| `stacks/_template/` | `$RULES/stack/` | Blank profile for other stacks. |

The rules describe architecture only: layers, contracts, errors, wiring, and tests. Business rules of a product belong in that project's own docs, and its terms go in `project-context.md`.

## Upgrade

Re-copy `core/` only:

```sh
cp -R "$KIT/backend/core/." "$RULES/core/"
```

`stack/` and `project-context.md` hold your project's content, so merge any changes to those by hand.
