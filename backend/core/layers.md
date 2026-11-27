# Architecture: Layers

Every feature of this API follows this layered architecture. Role names here are stack-neutral. `stack/stack-profile.md` maps each role to this project's directories, file names, and code shape.

Paths in these rules are relative to `.claude/rules/backend-architecture/`, unless they name a project source directory.

## Roles

| Role | Job |
|---|---|
| **Route** | Declares one HTTP endpoint: method, path, guards, and its Contracts. Passes the validated input to one Application Service method and returns the result. |
| **Contract** | Schema of what crosses the HTTP boundary: params, query, body, and response. The API contract and its OpenAPI document come from it. |
| **Application Service** | The only place for business orchestration: preconditions, authorization on a resource, transactions, Repository and Integration calls, events. Entry point for every HTTP and system action. |
| **Domain** | Entities, value objects, pure rules, and domain events. No I/O. |
| **Repository** | The persistence boundary for one table or aggregate. Reads and writes the database, and returns domain objects or typed rows. |
| **Integration** | Wraps one external system other than the app's own database: payments, email, CRM, calendar, LLM, cache, queue. Services depend on its interface. |
| **System entry** | A trigger that is not an HTTP request: a cron tick, a queue worker, a domain-event handler. |
| **Module** | Composition root of one context: creates its Repositories, Services, and Routes by hand, and registers them on the app. |
| **Infrastructure** | App-wide instances created once at boot: config, database connection, cache, queue producers, Integration clients, auth. Modules receive them and never create their own. |
| **Util** | Stateless technical helper with no business rule. |
| **Shared Contracts** | Typed errors, error codes, shared enums and ids. They depend on nothing. |

## Flow

```text
HTTP path:    Route -> Application Service -> Repository -> Database
                                           -> Integration -> External system
System path:  System entry -> Application Service -> Repository / Integration
```

Never skip a layer in either chain. A Route never reaches a Repository, and a System entry never does the work a Service owns.

## Who may call whom

| Caller | May call | Must never call |
|---|---|---|
| Route | One Application Service method per handler, guards | Repository, Integration, database client, another Route |
| Application Service | Repositories and Integrations through their interfaces, Domain, other Application Services, the event publisher, Utils | Routes, the database client directly |
| Repository | The database client, Domain, Utils | Application Service, Integration, Route |
| Integration | Its external system, Utils | Application Service, Repository, Route |
| System entry | Application Services. A domain-event handler that lives in the application layer works like one and may use Repositories | Repository (outside an application-layer handler), database client, Route |
| Domain | Other Domain code | Anything with I/O: database client, Integration, the HTTP framework |
| Module | Constructors of every role in its context, Infrastructure | Business logic |
| Util | Other Utils | Every other role |

- Contracts and Shared Contracts sit outside the chain, because "never skip a layer" governs calls, not shared types. Routes and Services may import Contract types. Repositories and Domain do not depend on HTTP Contracts.
- A Service depends on a Repository or Integration through an interface, so a test can replace it with a fake.
- Cross-context calls follow `core/cross-cutting.md`.

## What each layer must not contain

- **Route**: business rules, database or Integration access, or error responses built by hand. Errors go to the central error handler; see `core/contracts-and-errors.md`.
- **Application Service**: HTTP concerns: the request and reply objects, status codes, headers, cookies. It throws typed errors, and the error handler picks the status.
- **Repository**: business decisions. "Which rows" and "what to change" arrive as parameters from the Service.
- **Domain**: I/O, framework imports, environment reads, or the clock and random ids when a test needs to control them.
- **Integration**: business rules. It translates its SDK's failures into typed errors, so SDK error types never reach a Service.
- **Module**: anything but construction and registration.

## Building a feature

Create layers in this order, skipping any that already exist:

1. Shared Contracts: a new error code or shared enum
2. Domain
3. Database schema change and its migration
4. Repository
5. Integration, only for a new external system
6. Application Service
7. Contracts: input and response schemas
8. Route
9. Module wiring
10. Regenerate the OpenAPI document

Before creating a file in a role, Read the file that `stack/stack-profile.md` lists for that role. Path-scoped guidance loads only after a matching file has been touched, so it cannot shape the first file you write in a role.

After changing a Contract, a Route, or the database schema, run the generate commands in `stack/stack-profile.md`.

## Existing code that breaks these rules

Existing violations are not precedent. New code follows this file even when nearby code does not. Mention the violation to the user, and do not refactor it unless asked.
