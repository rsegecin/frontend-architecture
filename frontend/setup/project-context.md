# Project Context

Facts specific to this project. `setup-once.md` fills these in during the first session. Keep them current by hand. Facts belong here rather than in `core/`, so that upgrading the rules never overwrites them.

## Supported platforms

{{Platforms this project ships on, for example iOS only.}}

Investigate and change code for these platforms only. Other platform targets are out of scope unless the user explicitly asks.

## Domain glossary

{{Product terms, their synonyms, and the code they refer to.}}

When the user or the docs use one of these terms, treat it as this meaning. Do not invent a second feature for a synonym.

## API contract

{{Where the backend contract lives: live contract, local backend checkout, static snapshots.}}

Use these sources in order, and query them before assuming anything about the backend:

1. The live contract the app is tested against.
2. A local checkout of the backend, for implementation detail. Pull it first, because it may lag or be ahead of what is deployed.
3. Static snapshots, only when the others are unavailable. They drift.

## Related projects

{{Sibling repositories and embedded SDKs: their paths, and when to read them instead of guessing.}}

Treat this repository as the active codebase. Change related projects only when the task explicitly targets them.

## Logging and error reporting

- Logger: {{logger call}}
- Error-reporting facade: {{facade}}
- Failure-reason catalog: {{catalog location}}
- Coverage document: {{path, or none}}
