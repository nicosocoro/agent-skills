---
name: multi-agent-review
description: Review code changes with four subagents (prompt optimizer using gimme-prompt, code-review-planner, code-review-verifier, code-review-fixer) and apply the agreed fixes without user intervention. Compares the current branch against main by default. Use when the user invokes /multi-agent-review, says "multi-agent-review", or asks for a multi-agent code review.
disable-model-invocation: true
---

# multi-agent-review

Orchestrate four subagents in sequence. You are the orchestrator: you do not review or fix code yourself, and you do not ask the user anything until the final step.

## Workflow

1. **Resolve the scope**
   - **Repository**: the one the user names (`$ARGUMENTS`). Otherwise, every git repository among the session's working directory and additional directories whose current branch is not the base branch or has uncommitted changes. If none qualifies, tell the user and stop.
   - **Base**: the one the user names. Otherwise `main`, or `master` when `main` does not exist.
   - In each repository, run `git fetch origin --quiet`, then use `origin/<base>` when it exists, else the local `<base>`.
   - Compute `MERGE_BASE=$(git merge-base <base ref> HEAD)`. The diff command for every agent is `git diff $MERGE_BASE`, which covers the branch commits and the uncommitted changes.
   - If the diff is empty, skip that repository.
   - When there are several repositories, run steps 2 to 7 for one repository at a time, but give every agent all the repository paths so it can check cross-service contracts.

2. **Prepare**
   - Output folder: `~/Documents/reviews/<repo>/<branch with / replaced by ->/multi-agent/`. Create it.
   - Paths used below: `optimized-prompt.md`, `review.md`, `adjusted-plan.md`, `applied.md` inside that folder.

3. **Agent 0: optimize the prompt**
   - Run this step only if the `gimme-prompt` skill is available (listed among the available skills, or `~/.claude/skills/gimme-prompt/SKILL.md` exists) and the user gave instructions beyond the scope. Otherwise skip it.
   - Run a `general-purpose` subagent (foreground) with: the user's instructions verbatim, the absolute path of `optimized-prompt.md`, and this instruction: "Invoke the `gimme-prompt` skill with the instructions as input. You cannot ask the user questions: where the skill says to ask questions, state your assumptions instead. Write to the output path the optimized prompt plus a short list of assumptions, suggested improvements, and detected risks. Do not modify source code."
   - If `optimized-prompt.md` is missing, rerun once, then continue with the original instructions only and tell the user in the final step.

4. **Build the shared context**
   - Build a **shared context** block containing: the user's original instructions verbatim, the contents of `optimized-prompt.md` when it exists (the original stays as the source of truth if they conflict), every workspace path, the repository path, the base ref, the merge base, the diff command, the output of `git log --oneline $MERGE_BASE..HEAD`, `git status --short`, and the user rules (the full contents of `~/.claude/CLAUDE.md`, plus any rule the user stated in this conversation). Subagents do not see this conversation, so they only know what you pass them.

5. **Agent A: review and plan**
   - Run the `code-review-planner` subagent (foreground) with the shared context and the absolute path of `review.md`.
   - Check that `review.md` exists. If it is missing, rerun once, then stop and report the failure.

6. **Agent B: review and adjust the plan**
   - Run the `code-review-verifier` subagent (foreground) with the shared context, the absolute path of `review.md`, and the absolute path of `adjusted-plan.md`.
   - Check that `adjusted-plan.md` exists. If it is missing, rerun once, then stop and report the failure.
   - If its Fix plan is empty, skip step 7 and report that no fixes were needed.

7. **Agent C: apply**
   - Run the `code-review-fixer` subagent (foreground) with the shared context, the absolute path of `adjusted-plan.md`, and the absolute path of `applied.md`.
   - Check that `applied.md` exists. If it is missing, rerun once, then stop and report the failure.

8. **Present**
   - Read `applied.md` and show the user: what was applied, what was skipped, the verification result, and the pending items that need a decision.
   - Link the files (including `optimized-prompt.md` when it exists) and show `git status --short` so the user can see the changed files.
   - Do not commit or push.

## Launching a subagent

- Launch each subagent with the Agent tool, using its `subagent_type` (`general-purpose` for the prompt optimizer, then `code-review-planner`, `code-review-verifier`, `code-review-fixer`). The model comes from the frontmatter in `~/.claude/agents/<name>.md`.
- If that `subagent_type` is not available, launch a `general-purpose` subagent and put the body of `~/.claude/agents/<name>.md` at the top of the prompt, followed by the step inputs.

## Constraints

- Run the subagents one at a time. Each step needs the previous file.
- Do not edit `optimized-prompt.md`, `review.md`, `adjusted-plan.md`, or `applied.md` yourself.
- Do not commit, push, stash, reset, or switch branches.
