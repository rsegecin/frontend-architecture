# Project Context

Facts specific to this project. `setup-once.md` fills these in during the first session. Keep them current by hand. Facts belong here rather than in `core/`, so that upgrading the rules never overwrites them.

## Runtime and deployment

{{Node version, how the service is deployed (container, serverless, VM), how many instances run, and whether workers run with the API or apart.}}

Code that runs at boot, in a scheduler, or in a worker must be safe on that many instances.

## Domain glossary

{{Business terms, their synonyms, and the context or table they refer to.}}

When the user or the docs use one of these terms, treat it as this meaning. Do not invent a second feature for a synonym.

## API consumers

{{Who calls this API: web app, mobile app, other services. Where their code lives, and whether they use a generated client from the OpenAPI document.}}

Before changing a Contract, check what these consumers read. See `core/contracts-and-errors.md`.

## Authentication

{{How callers authenticate (session cookie, API secret, JWT), which routes are public, and where secrets come from.}}

## Database

- Engine and version: {{engine}}
- Schema and migrations: {{path}}
- Shared with other services: {{yes, which ones, or no}}
- Naming convention: {{camelCase or snake_case for tables and columns}}

## Related projects

{{Sibling repositories, shared packages, and SDKs: their paths, and when to read them instead of guessing.}}

Treat this repository as the active codebase. Change related projects only when the task explicitly targets them.

## Logging and error reporting

- Logger: {{logger call inside a request, and outside one}}
- Error reporting: {{facade or vendor, or none}}
