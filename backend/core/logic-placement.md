# Architecture: Where Logic Goes

Use this rule to pick the layer for a piece of logic.

## Quick decision

| Logic | Layer |
|---|---|
| Input shape: required fields, formats, ranges, enums, string trimming the client may skip | Contract |
| An invariant of one entity or value: an allowed state change, a valid range, a normalized form | Domain |
| A rule that needs stored data or several entities: uniqueness, ownership, plan limits, "already exists" | Application Service |
| What must happen atomically, and in which order | Application Service decides; `stack/stack-profile.md` says where the transaction opens |
| Which rows: filters, joins, sorting, pagination queries | Repository, with parameters from the Service |
| Turning a database error into a client conflict | Repository or the error handler, as the stack profile says |
| Calling an external API: auth, retries, timeouts, SDK quirks | Integration |
| Formatting and parsing with no business meaning | Util |
| Response shape | Contract; the Service returns data that fits it |

## Application Services

- One Service per capability of a context, such as `OrderService` or `OrderExportService`. Split a Service when its methods stop sharing dependencies.
- A method is one action: validate preconditions, load through Repositories, apply Domain rules, save, call Integrations, publish events.
- Every dependency arrives through the constructor. A Service never creates a Repository, an Integration, or a client, and never reads the environment. Config values arrive as constructor parameters.
- A Service throws typed errors. It never returns `null` to mean "not found" to a Route; it throws the not-found error.
- Work that may fail without failing the request, such as a cache warm-up or a notification, runs after the main write and catches and logs its own error. See `core/cross-cutting.md`.

## Domain

- Domain code is plain TypeScript: no framework, no database client, no environment, no network.
- An entity guards its own invariants. A change that breaks one throws the domain error, whoever calls it.
- Pure calculations used by several Services, such as a comparison, a price rule, or a time-slot computation, are Domain functions with unit tests.
- How much Domain a project keeps is a stack choice: rich entities, or plain types plus pure functions. The stack profile says which.

## Repositories

- One Repository per table or aggregate. It maps rows to Domain objects and back, so column naming and storage types stay inside it.
- It may return a typed read row for a join-heavy query instead of a Domain object, when no Domain behavior is needed.
- It holds no state between calls, and no cache unless the stack profile names one.

## Integrations

- One Integration per external system, behind an interface named for the capability, such as `PaymentGateway` or `CalendarProvider`, not for the vendor.
- The vendor implementation is named for the vendor: `StripePaymentGateway`. A no-op or console implementation for local runs is a second implementation of the same interface.
- It is created once at boot as Infrastructure, configured from the validated environment.

## System entries

- A cron tick, a queue worker, or an event handler only triggers: it calls an Application Service method, or is itself a thin function in the application layer.
- It is safe to run twice and on several instances: idempotent job ids, locks for "only one instance does this", and retries with backoff.

## Utils

- Pure technical helpers: concurrency limits, timezone math, id formats, text sanitizing.
- No injected dependencies, no Repository access, no business rule.
- Name them for what they do, such as `mapWithConcurrency` or `zonedParts`, never `*Service`.

## Anti-patterns

```text
Bad:  The Route checks whether the order belongs to the user, then calls the Repository.
Why:  Ownership is a business rule, and the Route skipped the Service.
Good: Route -> OrderService.get(accountId, id), which throws not-found for a foreign id.

Bad:  OrderService builds the Stripe client from process.env.
Why:  The Service creates Infrastructure and reads the environment.
Good: A plugin creates StripePaymentGateway at boot; the Module passes PaymentGateway to OrderService.

Bad:  OrderRepository.cancel() checks whether the order may still be cancelled.
Why:  A Repository makes no business decision.
Good: OrderService loads the order, the Domain checks the transition, the Repository saves it.
```

## Organizing code

Group code by context (a bounded area of the business), then by layer inside it. Never group the whole API by layer.

- **New capability in an existing context**: that context's folders.
- **New context**: create its folder only when adding its first file.
- **Technical capability used by several contexts** (email, cache, a vendor client): a named folder under the shared library directory, such as `lib/email/`.

Avoid `shared/`, `common/`, or `helpers/` buckets. Name the capability instead.
