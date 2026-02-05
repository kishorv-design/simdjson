You are an expert software engineer designing a debugging/triage exercise for a team you are mentoring.

### **Goal**

Analyze the provided codebase (it currently works correctly) and propose **100 candidate places** to introduce bugs that are realistic, subtle, and hard to discover through casual inspection and static analysis.

---

### **Critical constraints**

The bugs should not depend on understanding domain or business logic, but instead be largely mechanical correctness issues that can be identified from code semantics and general Python or OpenTelemetry knowledge.
The evaluation should focus only on core source files, not utility or helper modules.

Do not implement any changes yet. Only propose candidates.
Assume the evaluation agent will primarily do static analysis (it may not be able to run/compile the repo).
Prefer bugs in the core 20% of the codebase (hot paths / key modules), not obscure files.
Keep bugs plausible "normal engineering mistakes": off-by-one, race conditions, incorrect caching, error handling gaps, precision issues, boundary handling, config parsing quirks, timezones, retries, ordering, state leaks, security issues, etc.

**Additional difficulty constraints (must-follow):**

* At least 70% of proposed bugs must require reasoning across multiple functions or files, with cause and effect separated.
* Prefer bugs arising from implicit invariants, temporal ordering, or stateful assumptions rather than single-line mistakes.
* Avoid bugs that can be conclusively identified by inspecting a single line in isolation.
* Bugs should appear locally reasonable or intentional, and only appear incorrect when broader system behavior is considered.
* Avoid repeating bug patterns; if two bugs rely on the same underlying failure mode, they must occur in different subsystems or rely on different triggering conditions.

Avoid candidates that would be trivially discoverable via a single failing test or obvious runtime crash; if a bug would normally be caught by an existing test, note it as "likely too easy unless the direct test is removed."

---

### **Bug diversity targets**

Ensure the **100** span categories like:
correctness, reliability, concurrency, API contract drift, data integrity
caching/invalidation, pagination, edge-case input validation
time/date/timezone, numeric precision
resource leaks/cleanup, security issues
performance regressions (non-obvious)
whatever else that you deem relevant for a particular repo.

---

### **What you should do**

Map the codebase: identify major subsystems (entrypoints, core modules, data flow, I/O boundaries, async/concurrency, caches, DB access, API clients, background jobs, CLI, etc.).

Propose **100** specific bug-insertion opportunities, each with:

* ID: **B01…B100**
* Location: file path + function/class + approximate line range (best effort)
* Core relevance: why this is in the "core 20%" (what path/module makes it core)
* Bug type: e.g., off-by-one, stale cache, race condition, incorrect default, swallowed error, etc.
* Proposed change: minimal edit (describe in words, not a diff)
* Trigger conditions: when it manifests (inputs/timing/config/load/boundaries)
* Expected symptom: what would be observed (user/system behavior)
* Why it's hard: rare trigger, misleading signals, nondeterminism, cross-module assumption, etc.
* Static-analysis discoverability: how likely a strong reviewer would catch it without running code
* Suggested detection: test/monitoring/logging idea (unit/integration/property/load), even if Taiga won't run it

Rank all **75** by:

* Exercise value (educational + non-trivial)
* Stealth (subtlety / static-analysis resistance)
* Scorability (can be described as a clear rubric criterion: file/function + identifiable issue)

---

### **Output**

Write everything to **proposal.md**.

---

### **Iteration planning note**

When later creating iterations, form **up to 13 iterations**, each containing **exactly 8 bugs**, with no bug reused across iterations.
Prioritize combinations likely to yield **average evaluation scores below 0.5** and de-prioritize bugs that consistently score above 0.6.

---
