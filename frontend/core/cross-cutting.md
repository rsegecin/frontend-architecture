# Architecture: Cross-Cutting Rules

`.claude/rules/frontend-architecture/project-context.md` names this project's logger, error-reporting facade, and failure-reason catalog.

## Error reporting

When adding or extending production error reporting:

1. Report only through the project's facade. Feature code never calls the reporting vendor's SDK directly.
2. Every report carries a reason from the central catalog: no inline strings. Reuse an existing reason when the failure is the same event.
3. Add structured keys, such as ids and operation names. Attach no personal data beyond the user id.
4. If the project keeps a coverage document, update it.

Placement:

- **Use Cases and Services**: the primary place to report.
- **ViewModels**: only for view-specific failures that no Use Case owns.
- **Views**: never report directly.

Report unexpected failures that indicate a bug or an operational issue. Do not report expected client errors (4xx), user cancellation, validation failures, offline or transient connection errors, ended sessions, or debug-only test crashes.

## Logging

Start every log message with its origin in square brackets. The origin is the class or module emitting the log. Keep it concise and stable.

```text
Bad:  log('Error while uploading')
Good: log('[OrdersSyncService] Error while uploading')
```

## Dependency cycles

- If declaring a real dependency creates a cycle, restructure. Never hide or omit the dependency to silence the error.
- Moving a call into a helper does not remove the dependency, because the helper still runs on behalf of its owner.
- Lower layers never import higher layers. When a lower layer must cause a higher-level action, it only requests it:
  1. The lower layer raises a **signal**: a dependency-free Contract holding the request and its reason.
  2. A **coordinator** is a Service outside the cycle. It watches the signal and calls the Use Case.
  3. The app root View subscribes to the coordinator to keep it alive. Boot cannot start it, because the Use Case it calls may re-run boot.

For example, the transport detects an expired session and raises a signal. A coordinator then runs the sign-out Use Case, and the transport never depends on it.

## App boot

- Exactly one Use Case boots the app, and it starts everything in sequence: storage, session, settings, and Services. This is the boot Use Case.
- Boot is explicit. Creating the boot Use Case does not run it. Its entry method must be called.
- Cold start, sign-in, sign-out, and boot-failure retry all call that same entry method. No second path initializes the app.
- Boot runs in two phases, each with its own shell:
  1. The **boot shell** shows until the first boot succeeds.
  2. The **app shell** takes over and stays mounted across sign-in and sign-out.
- The flag that switches phases is set once and never resets.
- Shell Views trigger boot through their ViewModel, like any other View.
- Boot phases are required. A failed phase stops boot; do not soft-fail it into the app.
- Phase work uses cached data as a fallback for transient network errors when local data exists.
- On failure, show localized copy, never raw error text, and let retry call the same entry method.
- Boot phases read each dependency once. Anything running after boot subscribes to stay in sync.
- Global overlays live in the app shell only. Each is presentation-only and reads its own ViewModel.
