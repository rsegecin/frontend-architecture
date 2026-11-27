# Architecture Rules

Claude Code rules that give a project one layered architecture. Copy one kit into a project, and Claude Code follows it in every session.

| Kit | For | Chain |
|---|---|---|
| [`frontend/`](frontend/README.md) | Front-end apps: Flutter, React, Next.js | `View -> ViewModel -> Use Case -> Repository -> Data Source` |
| [`backend/`](backend/README.md) | HTTP APIs: Fastify | `Route -> Application Service -> Repository / Integration` |

Each kit has the same layout:

- `core/`: stack-neutral rules, always loaded.
- `setup/`: the first-session setup procedure and the project facts file.
- `stacks/<stack>/`: the profile that maps the core roles to one stack, plus rules that load when Claude touches matching files.

Each kit's README has the copy commands. The READMEs are never copied, because every `.md` file under `.claude/rules/` gets loaded.
