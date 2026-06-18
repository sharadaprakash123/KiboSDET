# Kibo SDET Challenge — Candidate Instructions

## Overview

Welcome to the **Kibo SDET Technical Assessment**.

You are inheriting a small mock Kibo fulfillment service (`Kibo.MockApi`) and an existing test suite (`Kibo.LegacyTests`). The current tests were written quickly by a junior QA engineer and are riddled with duplication, hardcoded values, and brittle timing assumptions.

**Your mission:** Demonstrate that you can transition from "manual / bloated" testing habits to building a scalable **Testing Platform / SDK** by refactoring the existing tests and extending them with modern engineering practices.

> **🤖 AI Tools Are Encouraged**
>
> Kibo values engineers who leverage AI effectively. You are **encouraged to use AI tools throughout this entire assessment** — GitHub Copilot, ChatGPT, Claude, Cursor, or any tool that's part of your workflow. We're not testing whether you can code without help; we're evaluating **how you use AI as a force multiplier**: your prompting strategy, your ability to critically evaluate output, and your judgment about what to keep, modify, or reject.
>
> Task 5 asks you to document your AI usage — so use AI freely and take notes as you go.

---

## Solution Structure

```
Kibo.SDET.Challenge.sln
├── src/
│   ├── Kibo.MockApi/              ← ASP.NET Core Web API (DO NOT MODIFY)
│   └── Kibo.TestingFramework/     ← Your reusable Testing SDK (BUILD HERE)
└── tests/
    └── Kibo.LegacyTests/         ← Inherited "bloated" tests (REFACTOR THESE)
```

| Project | Role |
|---|---|
| **Kibo.MockApi** | A simplified Kibo Fulfillment API. Exposes `POST /v1/orders` and `GET /v1/orders/{id}`. **Do not modify this project.** |
| **Kibo.TestingFramework** | An empty class library. This is where you build your reusable testing SDK — shared clients, builders, utilities. |
| **Kibo.LegacyTests** | The existing test suite. Refactor these tests to use your new framework. |

---

## Prerequisites

- [.NET 10 SDK](https://dotnet.microsoft.com/download/dotnet/10.0) (v10.0.x)
- An IDE or editor of your choice (Visual Studio, VS Code, Rider)
- Git
- **AI tools** — whichever you normally use (GitHub Copilot, ChatGPT, Claude, Cursor, etc.). There is no restriction on AI usage.

---

## Setup

```bash
# 1. Clone or unzip the repository
cd Kibo.SDET.Challenge

# 2. Restore & build
dotnet build

# 3. Run the Mock API (keep this running in a separate terminal)
dotnet run --project src/Kibo.MockApi

# 4. Verify the legacy tests pass (in another terminal)
dotnet test tests/Kibo.LegacyTests
```

> **Note:** The API listens on `http://localhost:5000`. All four legacy tests should pass (one takes ~6 seconds due to `Thread.Sleep`).

---

## API Quick Reference

### `POST /v1/orders`

Creates a new order.

| Header | Required | Description |
|---|---|---|
| `x-kibo-tenant` | **Yes** | Tenant identifier. Returns `401 Unauthorized` if missing. |
| `Content-Type` | Yes | Must be `application/json` |

**Request Body:**
```json
{
  "customerEmail": "customer@example.com",
  "lineItems": [
    {
      "productCode": "SKU-001",
      "quantity": 2,
      "unitPrice": 29.99
    }
  ]
}
```

**Response:** `201 Created` with the full `Order` object (includes generated `id`, `tenantId`, and `status: "Pending"`).

**Important behavior:** The order status automatically transitions from `"Pending"` → `"ReadyForFulfillment"` after exactly **5 seconds**.

### `GET /v1/orders/{id}`

Returns the current state of an order.

**Response:** `200 OK` with the `Order` object, or `404 Not Found`.

---

## Tasks

### Task 1: The Platform Shift

**Goal:** Eliminate duplication and build a reusable foundation.

1. **Analyze** `Kibo.LegacyTests/OrderTests.cs`. Identify every anti-pattern:
   - `HttpClient` created directly in each test method
   - Hardcoded `http://localhost:5000` URLs everywhere
   - `x-kibo-tenant` header logic copy-pasted into every test
   - Raw JSON strings built inline
   - No shared setup or teardown

2. **Extract** all shared logic into the `Kibo.TestingFramework` project:
   - Create a base API client class that manages `HttpClient` lifecycle, base URL configuration, default headers, and JSON serialization/deserialization.
   - Make the base URL configurable (environment variable, config file, or constructor parameter).

3. **Refactor** the legacy tests to use your new framework. The tests should be concise, readable, and free of duplication.

**Evaluation criteria:**
- DRY principle applied effectively
- Configuration is externalized (not hardcoded)
- Client lifecycle is properly managed
- Tests are readable and self-documenting

> 💡 *Feel free to use AI tools to help identify anti-patterns, scaffold your base client class, or generate boilerplate. If you do, note it in your prompt log — we want to see your process.*

**Goal:** One-line test data generation.

Manually constructing JSON payloads in every test is tedious, error-prone, and obscures test intent. Build a **Fluent Data Builder** in `Kibo.TestingFramework` that allows expressive, one-line test data construction.

**Example API (you design the exact interface):**

```csharp
var order = new OrderBuilder()
    .WithCustomerEmail("test@kibo.com")
    .WithItems(2)                         // generates 2 random line items
    .ForTenant("tenant-xyz")
    .Build();
```

**Requirements:**
- Sensible defaults (the builder should produce a valid order with zero configuration)
- Chainable / fluent API
- Support for custom line items and random data generation
- Placed in `Kibo.TestingFramework`

> 💡 *Builder patterns are well-suited to AI generation. If you use AI to scaffold your builder, the interesting part is how you refine the API design, handle defaults, and integrate randomization — that's where your judgment shows.*

---

### Task 3: Resiliency & Polling

**Goal:** Replace `Thread.Sleep()` with a smart polling utility.

The legacy test `GetOrder_AfterCreation_StatusBecomesReadyForFulfillment` uses `Thread.Sleep(6000)` to wait for the order status to change. This is:
- **Slow** — always waits the full 6 seconds even if the status changes after 5.01s
- **Flaky** — if the server is slow, 6 seconds may not be enough
- **CI-hostile** — wastes pipeline time on every run

Build a reusable **Polling / Wait-Until** utility in `Kibo.TestingFramework`:

```csharp
// Example usage (you design the API):
var readyOrder = await Poller.WaitUntilAsync(
    action: () => client.GetOrderAsync(orderId),
    condition: order => order.Status == "ReadyForFulfillment",
    interval: TimeSpan.FromMilliseconds(500),
    timeout: TimeSpan.FromSeconds(15)
);
```

**Requirements:**
- Configurable polling interval (default: 500ms)
- Configurable timeout (default: 15s)
- Throws a clear exception on timeout (include the last observed state in the message)
- Returns the result as soon as the condition is met (no unnecessary waiting)
- Generic — works with any async operation, not just orders

---

### Task 4: AI-Driven Edge Cases

**Goal:** Use GenAI to identify blind spots in test coverage.

Kibo encourages the use of generative AI tools. For this task:

1. **Use an LLM** (ChatGPT, Claude, Copilot, etc.) to analyze the API endpoints and generate **5 "destructive" or "edge-case" test scenarios**. Examples might include:
   - SQL injection attempts in the `x-kibo-tenant` header
   - Negative or zero pricing in line items
   - Extremely long strings in `customerEmail`
   - Empty `lineItems` array
   - Missing required fields
   - Unicode / special characters in various fields
   - Oversized payloads

2. **Implement** these 5 tests using your new `Kibo.TestingFramework` (using the builder, client wrapper, and any utilities you created).

3. **Document** the expected vs. actual behavior. If the API handles an edge case poorly (no validation, unexpected 500, etc.), note it as a **bug report** in a code comment.

**Evaluation criteria:**
- Creativity and thoroughness of edge cases
- Effective use of GenAI as a tool (not as a crutch)
- Tests are implemented using your framework (not copy-pasted raw HTTP)
- Clear documentation of findings

---

### Task 5: AI Prompt Log

**Goal:** Transparency in AI usage.

Create a file called `PROMPT_LOG.md` in the repository root. Document your AI usage **across all tasks** — not just Task 4. We want to understand your AI workflow: how you prompt, how you evaluate output, and what judgment calls you make.

For each significant AI interaction, document:

- The prompt text (or a summary)
- Which tool you used (ChatGPT, Copilot, Claude, etc.)
- Which task it was for
- How you applied the response (what you kept, what you changed, what you rejected and why)

Aim for **at least 5 entries** spanning multiple tasks. Quality matters more than quantity — we'd rather see 5 thoughtful entries showing iterative prompting and critical evaluation than 15 one-liners.

**Example format:**

```markdown
## Prompt 1 — Task 1 (Platform Shift)
**Tool:** GitHub Copilot Chat
**Prompt:** "Review this test class and list every anti-pattern related to HttpClient usage, 
URL hardcoding, and test isolation..."
**Outcome:** Copilot identified 5 of the 6 anti-patterns. It missed the lack of teardown/cleanup. 
I added that to my list manually. Used the output as a checklist while refactoring.

## Prompt 2 — Task 2 (Builder)
**Tool:** Claude
**Prompt:** "Generate a fluent builder pattern for an Order class with..."
**Outcome:** Used the generated skeleton but modified the `WithItems()` method to accept
a Func<LineItemBuilder, LineItemBuilder> for more flexibility. Rejected the suggestion
to use record types because the API expects mutable DTOs for deserialization.
```

> **What we're evaluating:** Not *how much* AI you used, but *how well*. A candidate who used AI for every task but critically evaluated and improved each output demonstrates stronger engineering judgment than one who avoided AI entirely.

---

### Task 6: Test Observability & Diagnostics

**Goal:** Make test failures easy to diagnose without re-running.

When a test fails in CI, the first question is always "what actually happened?" Raw assertion failures like `Expected: 201, Actual: 500` are nearly useless without context. Build **observability into your testing framework** so that failures are self-diagnosing.

Add the following capabilities to your `Kibo.TestingFramework` API client:

1. **Request/Response Logging** — Capture and expose the full HTTP request (method, URL, headers, body) and response (status code, headers, body) for every API call. This should be accessible when a test fails.

2. **Timing Capture** — Record the elapsed time for each API call. Expose this so tests can optionally assert on performance (e.g., "order creation should respond within 500ms").

3. **Correlation** — Each test or API call should have a unique identifier (e.g., a generated correlation ID sent via a custom header) that can be used to trace a test's requests through logs.

**Example usage (you design the exact API):**

```csharp
var response = await client.CreateOrderAsync(order);

// Timing is captured automatically
Console.WriteLine($"Request took {response.ElapsedMs}ms");

// Full diagnostics available for debugging
Console.WriteLine(response.RequestLog);  // POST /v1/orders { ... }
Console.WriteLine(response.ResponseLog); // 201 Created { ... }
```

**Requirements:**
- Logging should be **off by default** for clean test output, but easily enabled (environment variable, fluent toggle, or test fixture)
- Timing capture should always be active (it's cheap)
- Write at least **1 test** that asserts on response time to demonstrate the capability
- Placed in `Kibo.TestingFramework`

**Evaluation criteria:**
- Diagnostics are built into the framework, not bolted on per-test
- Timing assertions are included
- Design is toggleable / non-intrusive
- Demonstrates understanding of why test observability matters in CI/CD

> 💡 *HttpClient middleware / DelegatingHandler is one approach to request/response capture — AI tools can help you scaffold this pattern if you haven't used it before. Show us how you'd prompt for it and what you'd adjust.*

---

## Evaluation Criteria

| Dimension | What We're Looking For |
|---|---|
| **Code Reuse & Architecture** | Clean separation of concerns. Framework code is genuinely reusable, not just moved. |
| **Design Patterns** | Appropriate use of Builder, Factory, or Strategy patterns. Fluent APIs. |
| **Resiliency** | Polling replaces sleeping. Timeouts are configurable. Failures produce clear diagnostics. |
| **CI/CD Readiness** | Tests are deterministic, fast, and don't depend on hardcoded infrastructure. |
| **Observability** | Framework captures request/response details and timing. Failures are self-diagnosing. |
| **GenAI Fluency** | Thoughtful, pervasive use of AI tools across tasks — not blind copy-paste, but guided generation with critical evaluation and human judgment. Prompt log shows iterative refinement. |
| **Code Quality** | Consistent naming, proper async/await usage, XML docs where helpful, no warnings. |

---

## Time Expectation

This assessment is designed to take approximately **2–3 hours** with the use of AI tools. All six tasks are required. If you find yourself spending significantly more time, focus on quality over completeness — a well-designed framework with fewer tests is preferred over many tests with poor structure.

---

## Submission

1. Ensure the solution builds cleanly: `dotnet build`
2. Ensure your refactored tests pass: `dotnet test`
3. Include your `PROMPT_LOG.md` in the repo root
4. Push to a Git repository (GitHub, GitLab, etc.) and share the link, **or** zip the solution and email it

Good luck! 🚀
