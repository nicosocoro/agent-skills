---
name: review-branch
description: Reviews a git branch against a base branch (default main) in a local repo and writes a markdown report covering retrocompatibility issues, inconsistencies, bugs, refactors and code quality, each with a concrete fix. Use when the user asks to review a branch, compare a branch to main, or audit an MR/PR's changes before merging.
disable-model-invocation: true
---

# Review Branch

Inputs (ask only if missing): repo path, branch, base branch (default `main`). They may come in `$ARGUMENTS`.

## Workflow

```
- [ ] 1. Collect the diff
- [ ] 2. Read the context around every change
- [ ] 3. Check retrocompatibility, including other services
- [ ] 4. Check bugs, unbounded loops, inconsistencies and quality
- [ ] 5. Verify build and tests
- [ ] 6. Write the report
```

### 1. Collect the diff

Run from the repo root:

```bash
git fetch origin --quiet
git log --oneline <base>..<branch>
git diff --stat <base>...<branch>
git diff <base>...<branch> -- <source paths> <build and dependency files> > /tmp/review_main.diff
git diff <base>...<branch> -- <config, infrastructure and deployment paths> > /tmp/review_cfg.diff
git diff <base>...<branch> -- <test paths> > /tmp/review_test.diff
```

Pick the paths from the repository layout. Read the main diff and the config diff in full. Skim the test diff for coverage gaps and for tests that pin config values.

### 2. Read the context around every change

Don't review the diff alone. For each changed unit (class, module, function, component), read its callers, the branches that switch over values it adds or changes, the persistence and transport models it touches, and the place where it is wired or registered. Use the language's code navigation tools when available; otherwise use `Grep` and `Glob`.

### 3. Retrocompatibility checklist

- **Persistence**: new fields need defaults or must be optional. Do readers tolerate unknown fields? Do new enum or string values survive a rollback to the previous version?
- **API responses**: new values sent to clients. Check the client parsers (any client in the workspace) and the client version that first supports them. Is the feature gated by a flag or a version check?
- **Config**: renamed, removed or now-required keys. Does the new config still load for older readers?
- **Shared resources** (config, secrets, queues, topics, schemas, shared libraries): search for the resource name across every repo in the workspace and check how each consumer reads it on its default branch.
- **Dependencies**: confirm every bumped dependency version exists in its registry, and read its changelog or diff between the old and new version.
- **HTTP and message contracts**: compare paths, fields, status codes and event payloads against the other side's handler and models (open its repo if it's in the workspace).
- **Behavior changes** hidden in refactors: fallbacks, defaults, polling or batch scope, and states that used to be implicit.
- **Deploy order** across services, and whether config or infrastructure should ship separately.

### 4. Other categories

- **Bugs**: time boundaries (half-open intervals, midnight, timezone), cache TTL against the lifetime of the cached data, races between concurrent writes, partial failures in batched calls, silent empty or null results on invalid input, unhandled errors.
- **Unbounded loops and recursion**: see the section below. Check every change against it.
- **Inconsistencies**: the new feature behaving differently from its siblings (fallbacks, unknown states, validation style), config differences between environments, duplicated constants.
- **Refactors**: duplicated string keys that should be a constant or enum, positional arguments of the same type, logic gated twice, dead branches.
- **Code quality**: naming, unnecessary remote calls on hot paths, tests coupled to operational config, and any violation of the repository rules or the user rules.

### Unbounded loops and recursion

Flag any path that can call itself again without a guaranteed stop, directly or through other systems. Pay special attention when each iteration costs money: serverless functions, paid or metered APIs, LLM calls, cloud resources, messages and emails.

- **Recursion**: recursive functions without a reachable base case or a depth limit, especially over user input or external data.
- **Retries**: retry loops without a maximum number of attempts, backoff, or a check for errors that will never succeed.
- **Self-triggering events**: a function or job that writes to the storage, queue, topic, table or webhook that triggers it. For example: a function triggered by uploads that writes to the same bucket, a stream handler that updates the table it listens to, or a webhook handler that calls the API that fires the webhook again.
- **Mutual calls**: service A calls B, which calls A, with no hop limit or idempotency key.
- **Pagination and polling**: loops that trust a `next` token or a status field without a page cap, a timeout, or a check that progress is made.
- **Agents and LLM chains**: tool calls or agent steps that can call themselves again without a step limit or a budget.
- **Cost guards**: paid calls inside loops need a cap. Examples are a maximum number of iterations, a budget or rate limit, a timeout, reserved concurrency, or an alarm.

A loop like this is a 🔴 Blocker when each iteration costs money or can exhaust a shared quota. Otherwise it is 🟠 High.

### 5. Verify build and tests

Run the targeted tests for the touched areas. If the build fails for an external reason (such as an unpublished dependency), reproduce the fix in a throwaway worktree so the user's checkout stays untouched:

```bash
git worktree add -q --detach /tmp/review-wt <branch>
# apply a minimal fix there and run the targeted tests
git worktree remove --force /tmp/review-wt
```

Do not modify the reviewed branch.

## Report

Write to `~/Documents/reviews/<repo>/<branch with / replaced by ->.md` and tell the user the path. Use this template:

```markdown
# Review: `<repo>` — `<branch>` vs `<base>`

- Commits / diff size / what was verified (tests run, results)

## Summary
One paragraph: overall assessment, blockers, main risks.

| # | Severity | Area | Title |
|---|----------|------|-------|

## 🔴 Blockers
### <ID>. <Title>
Problem with `file:line` and evidence. Impact. **Fix** with a code snippet.

## 🟠 High
## 🟡 Medium
## 🟢 Low

## Retrocompatibility checks that pass
## Suggested merge plan
```

Severity:
- 🔴 Blocker: doesn't build, breaks production or another service, or can loop while it costs money.
- 🟠 High: user-facing bug, data loss, or an unbounded loop with no direct cost.
- 🟡 Medium: latent bug, behavior change, or inconsistency.
- 🟢 Low: quality, refactor, or nit.

ID prefixes: `B` build, `R` retrocompat, `G` bug, `L` unbounded loop or recursion, `I` inconsistency, `Q` quality/refactor.

Every finding needs evidence (file:line, command output, or the other repo's code) and a concrete fix. Mark anything unverifiable as "verify".
