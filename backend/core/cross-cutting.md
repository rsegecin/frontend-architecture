# Architecture: Cross-Cutting Rules

`.claude/rules/backend-architecture/project-context.md` names this project's logger, error reporting, and deployment facts.

## Configuration

- One module reads `process.env`. It validates every variable with a schema at boot, and the app refuses to start when one is invalid.
- A new variable goes into that schema and into `.env.example`, in the same change.
- Config reaches a Service as a constructor parameter. Domain and Services never read the environment.

## Composition

- Dependencies are wired by hand in each Module. There is no container.
- Infrastructure is created once, at boot, by the plugins that own it, and closed on shutdown.
- A Module receives Infrastructure from the app. It never creates a database client, a cache connection, or a vendor client of its own.
- Boot order matters: config, then connections, then Integrations, then Modules, then whatever listens to Modules (event wiring, schedulers). `stack/stack-profile.md` points to the file that holds the order.

## Authentication and authorization

- Authentication happens at the Route boundary, in a guard that runs before the handler. An unauthenticated request never reaches a Service.
- A guard that only checks a secret runs before body parsing and validation, so an anonymous caller cannot make the API parse a large body.
- Compare secrets and tokens in constant time.
- Authorization on a resource ("is this order yours?", "may this admin change that user?") happens in the Service, which knows the resource. A role guard on the Route only answers "may this caller use this endpoint at all".
- Every query for a caller's data filters by the caller's id in the Repository call. Never load by id alone and check ownership afterwards in memory.

## Transactions and concurrency

- A write that changes several rows, or a read-check-write that two requests could race, runs in one transaction.
- When concurrent requests could break an invariant (two "current" records, a limit exceeded), lock before reading, in a consistent order, to avoid deadlocks.
- Keep slow external calls outside the transaction. Run best-effort side effects after it commits.

## Cross-context calls

A context reaches another context in one of three ways:

1. Through that context's Application Service, when the action is its business.
2. Through that context's Repository interface, narrowed to the methods it needs, for plain reads.
3. Through a domain event, when the other context reacts to something that happened, and the caller must not depend on the reaction.

Never import another context's Routes, and never copy its Domain rules.

## Background and best-effort work

- Fire-and-forget work always has a `.catch` that logs. An unhandled rejection is a bug.
- Jobs are idempotent and safe on several instances. See `core/logic-placement.md`.
- A queue or scheduler started at boot is stopped in the shutdown hook.

## Logging

- Log through the framework's logger: the request logger inside a request, the app logger elsewhere. Never `console.log` in application code.
- Log structured: the context object first, a short stable message second.
- Never log secrets, tokens, passwords, or full request bodies with personal data.

```text
Bad:  console.log('failed to send email ' + error)
Good: app.log.warn({ err, orderId }, 'order email: send failed')
```

## Error reporting

- Report unexpected failures only: 5xx, failed jobs, broken invariants. Do not report validation errors, 4xx, or expected external timeouts that are retried.
- Report through the project's logger or facade named in `project-context.md`, never a vendor SDK called from feature code.

## Dependency cycles

- If declaring a real dependency creates a cycle, restructure. Never hide or omit the dependency to silence the error.
- Lower layers never import higher layers. When a lower layer must cause a higher-level action, it raises a domain event, or the Service passes the needed data down as a parameter.
