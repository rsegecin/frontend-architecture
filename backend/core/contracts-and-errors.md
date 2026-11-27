# Architecture: Contracts and Errors

## HTTP Contracts

- The Contract schemas are the API contract. The OpenAPI document is generated from them. Never write it by hand, and never document routes in a README or in comments, because those drift.
- Every Route declares a schema for each input it reads (params, query, body) and for every success status it returns.
- The response schema decides what leaves the API. A field the schema does not list is never sent, so a Service can return a richer object without leaking it.
- Input validation lives in the Contract. A Service may still check a rule that needs data; it never repeats a format check the Contract already made.
- After changing a Contract or a Route, regenerate the OpenAPI document with the command in `stack/stack-profile.md`.

### Changing a Contract

Clients depend on the contract, so a change can break them.

| Change | Safe |
|---|---|
| Add an optional input field, or a response field | Yes |
| Add a new Route | Yes |
| Remove or rename a field, change its type, make an input required, tighten a format | No: ask the user first |

### Query values

- Query strings arrive as text. Coerce numbers explicitly, and parse booleans from `"true"` and `"false"`. A generic boolean coercion reads `"false"` as true.
- An unescaped `+` in a query string arrives as a space. Handle it in the schema for any value that may contain `+`.

## Pagination

Every list Route is paginated, with one request and response shape across the API. `stack/stack-profile.md` names it. A new list never invents another shape.

## Typed errors

- Services, Domain, Repositories, and Integrations throw typed errors. Nothing outside the central error handler builds an HTTP error response.
- The central error handler maps each error type to a status, a stable `code`, and one body shape. A new error type gets its mapping there.
- Every error carries a machine-readable `code`. Clients branch on the code, never on the message.
- Contract validation failures are answered by the error handler with the validation status and the failing fields.
- Expected client errors (4xx) are not logged as errors. Unexpected errors are logged once, with the request context, and answered with a generic 500 body. Never send a stack trace, an SQL message, or an internal message to the client.
- A database error that means a client conflict, such as a unique violation, becomes the conflict error. The stack profile says whether the Repository or the error handler translates it.

| Error kind | Typical status |
|---|---|
| Contract validation | 400 or 422, as the stack profile says |
| Domain invariant broken | 422 |
| Not authenticated | 401 |
| Authenticated but not allowed | 403 |
| Resource not found, or not visible to the caller | 404 |
| Conflict with current state: duplicate, lock busy | 409 |
| Rate limited | 429 |
| External system failed | 502 or 503 |

A resource that exists but belongs to someone else answers 404, not 403, so ids cannot be probed.

## Database schema

- Every schema change ships with a migration in the project's migrations directory.
- A migration must apply to a database that already has data: new columns are nullable or have a default, and data moves happen in the migration.
- Never edit a migration that has run in a shared environment. Add a new one.
- After a schema change, regenerate the database client or types with the command in `stack/stack-profile.md`.
- Table and column naming follows the convention already in the schema. The stack profile names it.
