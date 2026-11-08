---
name: review-plan
description: Review an implementation plan file to find improvements, inconsistencies, possible bugs, and retrocompatibility issues, and write a markdown file with the findings. Use when the user invokes /review-plan or asks to review a plan file.
disable-model-invocation: true
---

# review-plan

Review a plan file against the actual codebase to identify **possible bugs**, **retrocompatibility issues**, **inconsistencies**, and **improvements**, and write a markdown file with the findings.

## Workflow

1. **Read the plan**
   - Read the target plan markdown file provided by the user (`$ARGUMENTS`).
   - Identify every repository, module, file, endpoint, data model, storage key or table, config shape, event or queue, and feature flag referenced in the plan.

2. **Inspect the codebase thoroughly**
   - Read the referenced files and search for all callers, implementations, and tests across the affected repositories (using `Read`, `Grep`, and `Glob` in parallel).
   - Trace every symbol, enum, data model, cache entry, route, event, and flag end-to-end across service and client boundaries.

3. **Analyze across the 4 required dimensions**
   - **Possible Bugs**:
     - Exhaustive `switch`/`match` statements or allowlists missing new enum or union values.
     - Unhandled null/empty/error states, race conditions, boundary conditions (timezones, midnight wrap-around, inclusive vs. exclusive intervals, TTL mismatches).
     - Missing wiring or registration (dependency injection, routers, handlers), mappers dropping fields, context lost in background jobs, or entry points that bypass the new flow.
     - Missing assets, package or manifest registrations, or link parameters.
     - **Unbounded loops and recursion**, especially where each iteration costs money (see below).
   - **Retrocompatibility Issues**:
     - Rolling-deploy and cross-service deployment order hazards (new required fields without defaults in stored data, messages, or HTTP contracts).
     - Legacy storage keys, internal endpoints, or stored state formats that could fail to deserialize or collide.
     - Behavior when older clients interact with updated backends (or vice versa), and whether feature flags need a minimum client version.
     - Shared libraries, config, schemas, or queues that other services in the workspace also consume.
   - **Inconsistencies**:
     - Contradictions between different sections of the plan (e.g., fallback values, endpoint paths, flag behavior, or responsibilities that live in a different layer or module in the actual codebase).
     - Mismatches with the user rules in `~/.claude/CLAUDE.md` and the repository rule files (`CLAUDE.md`, `AGENTS.md`, and the rule files the repository keeps, such as `.cursorrules` and `.cursor/rules/*.mdc`).
   - **Improvements**:
     - Simpler or safer ways to achieve the goal using existing abstractions.
     - Failure isolation (fail-closed/fail-open semantics, circuit breakers, timeouts, bulkheads).
     - Missing test cases or edge-case scenarios that should be added to the test plan.

### Unbounded loops and recursion

Flag any step that can call itself again without a guaranteed stop, directly or through other systems. Pay special attention when each iteration costs money: serverless functions, paid or metered APIs, LLM calls, cloud resources, messages and emails.

- Recursive functions without a reachable base case or a depth limit.
- Retries without a maximum number of attempts, backoff, or a check for errors that will never succeed.
- Functions or jobs that write to the storage, queue, topic, table or webhook that triggers them.
- Services that call each other with no hop limit or idempotency key.
- Pagination or polling without a page cap, a timeout, or a check that progress is made.
- Agent or LLM chains without a step limit or a budget.

For each one, the fix must name the guard: a maximum number of iterations, a separate output target, an idempotency key, a budget or rate limit, a timeout, reserved concurrency, or an alarm. Treat it as a blocker when the loop costs money.

4. **Create the findings markdown file**
   - Write the review to a markdown file next to the plan (e.g., `<plan_name>_review.md` in the same directory as the plan, unless the user specifies another path).
   - For every finding, cite the exact file paths, functions or classes, and concrete code snippets, and provide a clear **Fix** or **Recommendation**.

## Output Template

Use this structure for the generated markdown file:

```markdown
# <Plan Title> — Plan Review

Review of `<plan_path>`, organized into **Possible Bugs**, **Retrocompatibility Issues**, **Inconsistencies**, and **Improvements**.

---

## 1. Possible Bugs

### A. <Short descriptive title> (<Affected Service/Package & Plan Section>)
- **Issue:** <Detailed explanation referencing exact files, methods, and line numbers>
- **Fix:** <Concrete actionable fix>

---

## 2. Retrocompatibility Issues

### A. <Short descriptive title> (<Affected Service/Package & Plan Section>)
- **Issue:** <Rollout, serialization, cache, or client-version compatibility risk>
- **Fix:** <Concrete actionable fix>

---

## 3. Inconsistencies

### A. <Short descriptive title> (<Plan Sections>)
- **Issue:** <Contradiction within the plan or between the plan and the actual codebase architecture>
- **Fix:** <How to align them>

---

## 4. Improvements

### A. <Short descriptive title> (<Affected Service/Package>)
- **Suggestion:** <Architectural, resilience, or testing improvement>
```
