# Common review checks

These are defects that reviews keep raising in app and backend code. The brief includes the checks that apply to the files the change touches. For each one, the review says whether it applies and whether the change passes it.

**Changing the list.** Use `.intent/review-checks.md` (commit it to share it with the project) or `~/.claude/intent/review-checks.md` (only yours). In either file:
- **Add a check:** one per line, as `- <id>: <check>`.
- **Switch one off:** a line `off: <id>`, for example `off: ui-tokens`. It switches off a common or project check, but only for runs that read that file.
- **Change one:** switch it off and add your own version under a new id.

Findings from personal checks are posted like any other finding, without naming the check or the file.

A check never overrides the ticket: when the brief lists the behaviour as deliberate, the check doesn't apply. Skip any section whose stack the change doesn't use.

## Async, locks and waits
- `release-in-finally`: busy flags, spinners, disabled buttons and navigation locks are released in `finally`. A throw before the `try` must not leave them on forever.
- `bounded-waits`: every wait that blocks the UI has a limit. If the HTTP client has no timeout, add one that aborts the request, or release the lock after N seconds.
- `timeout-cancels`: a timeout cancels the work (abort the request, and clear the timer in `finally`). It doesn't just stop waiting.
- `unknown-after-timeout`: after a timeout on a call with server-side effects, the outcome is unknown, not failed. Settle it by asking the source of truth.
- `no-double-action`: an action can't run twice. It is disabled while pending, and there is a single source of truth for "pending" (not a ref and a flag that can drift apart).
- `escape-while-pending`: close and back stay usable while something is pending, or the lock has a limit. No invisible modal or stuck route is left behind.
- `toctou`: act on exactly what you validated. Pass the checked object forward instead of selecting it again later.
- `listener-isolation`: in listener fan-outs (event emitters, SDK callbacks, sets of subscribers), each call is wrapped in a try/catch that logs. One throwing subscriber must not stop the others.

## Data and state
- `invalidate-what-changed`: a mutation invalidates or updates everything it changes, both on success and on the uncertain error path. Use the project's existing invalidation helpers.
- `cache-merge`: a mutation writes only the fields it changed into the client cache, then re-syncs the rest.
- `optimistic-rollback`: an optimistic UI change is rolled back when the action fails. Never leave on screen something that didn't happen.
- `publish-complete-state`: state is published only once it is complete. Other readers (cache, public profile, CDN) must not see intermediate states. URLs are returned only for objects that exist.
- `guard-optional-access`: guard every indexed or optional access that can be missing at runtime (`array[0]`, optional API fields), especially when the type checker doesn't enforce it.
- `request-metadata-source`: request metadata the runtime can override (content type, headers) comes from one source. Check what actually arrives, not what you think you send.

## Errors and analytics
- `honest-error-classes`: a network outage, a timeout and a real bug never share the same error reason. Detect the actual cause instead of defaulting everything to "network" or "failed".
- `offline-detection`: detect offline through the platform's network status at failure time, not by parsing error messages, which differ per platform.
- `analytics-complete-data`: an event or screen view fires only after the data it reports has resolved.

## Rendering and performance (React-style UI code)
- `no-derived-state-in-effects`: state derived from props or other state is computed during render, not synced through effects or refs read during render.
- `stable-memo-deps`: memoization actually holds. Its dependencies are stable, and inline objects or functions in the deps are hoisted or memoized.
- `narrow-subscriptions`: don't subscribe to a whole store or query just to read one value or call one method. Read it on demand.
- `no-reread-known-state`: don't re-read platform or system state (native bridges, OS settings) the caller already has. Pass it in.
- `cheap-reporting-effects`: an effect that reports to external SDKs on every dependency change is cheap or batched (throttle, debounce, or send only the changed keys).
- `ui-tokens`: spacing, colours and typography come from the design tokens. Where no token exists, follow the project's convention for missing tokens.

## Scope and structure
- `reuse-existing`: reuse the existing helper or shared path instead of re-implementing it. Anything two places derive has one source of truth.
- `root-fix`: prefer the fix at the right layer (backend, shared helper) over a local workaround for another component's bug.
- `no-unneeded-code`: every added line serves the task. Flag extras such as speculative options, abstractions with one caller, wrappers that only forward, and config nobody sets.
- `no-dead-code`: no unused imports, variables, parameters, unreachable branches or leftovers from earlier attempts. Every symbol you touched still has a caller.
- `no-duplication`: before adding a helper, constant, style or rule, look for an existing one. Two copies of the same rule become one.
- `comments-and-tests-in-place`: a comment describes the code right below it, and a test sits under the `describe` it belongs to.

## Backend and infrastructure
- `cdn-404`: a CDN caches 404s too. Never let a client request a URL before its object exists, and remember that query-string variants are separate cache entries.
- `stream-pipeline`: streams use `pipeline()` from `node:stream/promises` and await it. `.pipe()` neither waits for "finish" nor propagates errors, and an unhandled stream error crashes the process.
