# Prompt Log

This log documents AI usage across the six tasks of the Kibo SDET Challenge.
The `Kibo.TestingFramework` (ApiClient, OrderBuilder, Poller, ObservabilityHandler,
ApiResponse) and the refactored/new test files were scaffolded with Claude in an
iterative session, then reviewed and adjusted against the actual Kibo.MockApi
contract (Order/LineItem shapes, the `x-kibo-tenant` header requirement, and the
5-second Pending -> ReadyForFulfillment transition). Each entry below describes
what was asked, what came back, and what was reviewed/adjusted before it went
into the final code.

---

## Prompt 1 — Task 1 (Platform Shift)

**Tool:** Claude

**Prompt:** Asked Claude to review the legacy `OrderTests.cs` (per-test
`HttpClient`, hardcoded `http://localhost:5000`, duplicated `x-kibo-tenant`
header logic, inline JSON, `Thread.Sleep`) and scaffold a reusable `ApiClient`
for `Kibo.TestingFramework` that fixes each anti-pattern, with the base URL
externalized via an environment variable.

**Outcome:** Claude produced `ApiClient` as an `IDisposable` wrapper around a
single `HttpClient`, reading the base URL from `KIBO_API_BASE_URL` with a
`http://localhost:5000` fallback, and centralizing the `x-kibo-tenant` header
so individual tests no longer set it manually. I reviewed this against the
five anti-patterns listed in the candidate instructions and confirmed each one
is addressed: the `using var client = new ApiClient()` pattern in the
refactored tests gives proper lifecycle management, and the header logic now
lives in one place (`SendAsync`'s `effectiveTenant` resolution) instead of
being copy-pasted per test.

---

## Prompt 2 — Task 2 (Builder)

**Tool:** Claude

**Prompt:** Asked Claude to generate a fluent builder for `Order` (with
`CustomerEmail`, a list of `LineItem` (`ProductCode`/`Quantity`/`UnitPrice`),
and a tenant id), supporting `WithCustomerEmail()`, `WithItems(int count)` for
randomized line items, `ForTenant()`, and `Build()`, with sensible defaults so
`new OrderBuilder().Build()` produces a valid order with zero configuration.

**Outcome:** Claude produced `OrderBuilder` with the requested fluent surface
plus a randomized line-item generator drawing from a small fixed pool of SKUs
and a bounded price range (1.00-199.99). I reviewed the defaults — confirmed
`Build()` falls back to one random item if `WithItems`/`WithLineItem` was never
called — and added two methods beyond the original scaffold:
`WithLineItem(productCode, quantity, unitPrice)` and `WithNoItems()`. These
were needed for Task 4's edge-case tests (negative pricing, empty cart), which
require explicit control over individual line items that pure randomization
can't express.

---

## Prompt 3 — Task 3 (Polling)

**Tool:** Claude

**Prompt:** Asked Claude for a generic `Poller.WaitUntilAsync<T>` static method:
takes an async `Func<Task<T>>` action and a `Func<T, bool>` condition, polls on
a configurable interval (default 500ms) until the condition is true or a
configurable timeout (default 15s) elapses, returns as soon as the condition is
met, and throws a `TimeoutException` containing the last observed value on
timeout.

**Outcome:** Claude produced the loop structure (`action()` → check
`condition()` → `Task.Delay` → repeat) using a `CancellationTokenSource(timeout)`
as the single timeout mechanism, so the overall timeout is enforced even if a
particular poll attempt is slow. I reviewed the timeout-message formatting —
the initial version would have used `.ToString()` on the last result, which
for an `Order`-shaped object would just print the type name — and confirmed it
instead uses `JsonSerializer.Serialize(last)` via the `Describe<T>` helper, so
`GetOrder_AfterCreation_StatusBecomesReadyForFulfillment` would produce a
useful timeout message (e.g. showing `"status":"Pending"`) if the polling ever
failed.

---

## Prompt 4 — Task 4 (Edge Cases)

**Tool:** Claude

**Prompt:** Gave Claude the API contract — `POST /v1/orders` (body:
`customerEmail`, `lineItems[]` with `productCode`/`quantity`/`unitPrice`;
requires `x-kibo-tenant` header, returns 201) and `GET /v1/orders/{id}`
(200/404) — and asked for 5+ destructive/edge-case test scenarios that probe
missing validation, security issues, and boundary conditions.

**Outcome:** Claude suggested SQL injection in the tenant header, an XSS/script
tag in `customerEmail`, an empty `lineItems` array, a negative `unitPrice`, an
oversized `customerEmail`, and a request body missing `lineItems` entirely. I
implemented six of these in `EdgeCaseTests.cs`, each using `OrderBuilder` and
`ApiClient` (the SQLi case via `ApiClient.CreateClientWithTenant`, the
missing-field case via `CreateOrderRawAsync` with a hand-written JSON body).
For each test I ran it against the running MockApi, observed it return
`201 Created` in every case (i.e. no input validation at all), and wrote that
up as a BUG REPORT in the XML doc comment above each test — documenting
expected behavior (400 Bad Request / 413), actual behavior (201 Created,
payload accepted verbatim), the risk, and a recommendation.

---

## Prompt 5 — Task 6 (Observability)

**Tool:** Claude

**Prompt:** Asked Claude how to build an `HttpClient` `DelegatingHandler` that
adds a correlation ID to every outgoing request, times each call, captures a
human-readable request/response log, and surfaces all of that back to the
caller — toggleable via an environment variable for console output.

**Outcome:** Claude produced `ObservabilityHandler` with the `SendAsync`
override pattern (generate correlation ID → build request log → `Stopwatch`
around `base.SendAsync` → build response log → optionally print to console).
The key design point I reviewed was *how diagnostics get back to the caller*:
rather than a shared dictionary keyed by correlation ID (which isn't safe
under xUnit's parallel test execution), the handler attaches
`X-Kibo-Correlation-Id`, `X-Kibo-Elapsed-Ms`, `X-Kibo-Request-Log`, and
`X-Kibo-Response-Log` directly onto the `HttpResponseMessage`, and `ApiClient`
reads them back off when building `ApiResponse<T>` — no shared mutable state.
I also confirmed the `Truncate()` step (100-char body limit) keeps logs
readable for larger payloads, and that `EnableConsoleLogging` defaults from
`KIBO_TEST_LOGGING` but is also a settable property on `ApiClient` for
per-test opt-in (see `EnableConsoleLogging_PrintsRequestAndResponseDiagnostics`
in `ObservabilityTests.cs`).

---

## Notes on Overall AI Workflow

Across all five tasks, the workflow was: ask Claude for a scaffold of the
underlying *pattern* (HttpClient wrapper, fluent builder, generic poller,
DelegatingHandler) rather than asking it to write tests against the live API
directly, then adapt the result to Kibo.MockApi's actual contract and verify
it against the running API. The recurring review point across Tasks 1, 3, and
6 was avoiding shared/static mutable state (a static HttpClient, `.ToString()`
on a complex object in an exception message, a static diagnostics dictionary)
in favor of instance-owned state and self-contained response objects.
