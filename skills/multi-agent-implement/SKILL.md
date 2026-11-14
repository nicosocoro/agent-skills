---
name: multi-agent-implement
description: Implement an existing plan with three subagents (plan-implementer, implementation-reviewer, implementation-arbiter) without user intervention, then report the review findings left for the user and the decisions worth knowing. Use when the user invokes /multi-agent-implement, says "multi-agent-implement", or asks for a multi-agent implementation of a plan.
disable-model-invocation: true
---

# multi-agent-implement

Orchestrate three subagents in sequence. You are the orchestrator: you do not implement, review, or decide yourself, and you do not ask the user anything until the final step.

## Workflow

1. **Prepare**
   - Take the plan from the user's message (`$ARGUMENTS`): a plan file path, or a plan folder created by `multi-agent-plan` (use its `final-plan.md`, else `plan.md`). If no plan was given, look for the most recently modified `.claude/plans/*/final-plan.md` in the primary workspace root and confirm it with the user. If there is none, ask for the plan and stop.
   - Output folder: the plan's folder when it is under `.claude/plans/<slug>/`. Otherwise create `<primary workspace root>/.claude/plans/<slug>/` with a short kebab-case slug for the plan. The primary workspace root is the repository the plan mainly affects, or the current working directory when unclear.
   - Paths used below: `implementation.md`, `implementation-review.md`, `applied.md` inside that folder.
   - In each affected repository, record the **baseline**: `git rev-parse HEAD`, the current branch, and `git status --short`. Files already modified or untracked before step 2 are pre-existing changes; list them so the agents ignore them. The diff command for every agent is `git diff <baseline HEAD>` plus the untracked files reported by `git status --short` that are not pre-existing.
   - Build a **shared context** block containing: the user's instructions verbatim, the absolute plan path, every working directory and additional directory of this session, the baseline per repository, the diff command, the pre-existing changes, and the user rules (the full contents of `~/.claude/CLAUDE.md`, plus any rule the user stated in this conversation). Subagents do not see this conversation, so they only know what you pass them.

2. **Agent A: implement**
   - Run the `plan-implementer` subagent (foreground) with the shared context and the absolute path of `implementation.md`.
   - Check that `implementation.md` exists. If it is missing, rerun once, then stop and report the failure.
   - If the diff is empty, stop and report that nothing was implemented, with the reason from `implementation.md`.

3. **Agent B: review**
   - Run the `implementation-reviewer` subagent (foreground) with the shared context and the absolute paths of `implementation.md` and `implementation-review.md`.
   - Check that `implementation-review.md` exists. If it is missing, rerun once, then stop and report the failure.

4. **Agent C: decide and apply**
   - Run the `implementation-arbiter` subagent (foreground) with the shared context and the absolute paths of `implementation.md`, `implementation-review.md`, and `applied.md`.
   - Check that `applied.md` exists. If it is missing, rerun once, then stop and report the failure.

5. **Present**
   - Read `implementation.md` and `applied.md` and show the user:
     - a short summary of what was implemented,
     - **Pending findings**: the review findings that were not applied and need the user's decision, each with its severity, the problem, and the proposed fix,
     - **Decisions worth knowing**: the notable decisions taken by Agent A (deviations from the plan, assumptions, choices the plan left open) and by Agent C (findings rejected or applied differently than proposed),
     - the verification result (typecheck, tests, analyzer),
     - `git status --short` of each affected repository.
   - Link the plan and the three files so the user can inspect the intermediate steps.
   - Do not apply the pending findings. Do not commit or push. Wait for the user.

## Launching a subagent

- Launch each subagent with the Agent tool, using its `subagent_type` (`plan-implementer`, `implementation-reviewer`, `implementation-arbiter`). The model comes from the frontmatter in `~/.claude/agents/<name>.md`.
- If that `subagent_type` is not available, launch a `general-purpose` subagent and put the body of `~/.claude/agents/<name>.md` at the top of the prompt, followed by the step inputs.

## Constraints

- Run the subagents one at a time. Each step needs the previous file and the code left by the previous agent.
- Do not edit the plan, `implementation.md`, `implementation-review.md`, or `applied.md` yourself.
- Do not modify source code yourself.
- Do not commit, push, stash, reset, or switch branches.
