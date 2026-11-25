---
name: multi-agent-implement
description: Implement an existing plan or a task given as a prompt with up to four subagents (prompt optimizer using gimme-prompt when the task comes from a prompt, plan-implementer, implementation-reviewer, implementation-arbiter) without user intervention, then report the review findings left for the user and the decisions worth knowing. Use when the user invokes /multi-agent-implement, says "multi-agent-implement", or asks for a multi-agent implementation of a plan or task.
disable-model-invocation: true
---

# multi-agent-implement

Orchestrate up to four subagents in sequence. You are the orchestrator: you do not implement, review, or decide yourself, and you do not ask the user anything until the final step.

## Workflow

1. **Prepare**
   - Determine the **input mode** from the user's message (`$ARGUMENTS`):
     - **Plan mode**: the message gives a plan file path, or a plan folder created by `multi-agent-plan` (use its `final-plan.md`, else `plan.md`). If no task text was given either, look for the most recently modified `.claude/plans/*/final-plan.md` in the primary workspace root and confirm it with the user. If there is none, ask for the plan and stop.
     - **Prompt mode**: the message is a task description with no plan. Nothing is asked of the user. The brief for the agents is the task description, improved by step 2 when it runs.
   - Output folder: in plan mode, the plan's folder when it is under `.claude/plans/<slug>/`. Otherwise (and always in prompt mode) create `<primary workspace root>/.claude/plans/<slug>/` with a short kebab-case slug. The primary workspace root is the repository the plan mainly affects, or the current working directory when unclear.
   - Paths used below: `optimized-prompt.md` (prompt mode only), `implementation.md`, `implementation-review.md`, `applied.md` inside that folder.
   - In each affected repository, record the **baseline**: `git rev-parse HEAD`, the current branch, and `git status --short`. Files already modified or untracked before step 4 are pre-existing changes; list them so the agents ignore them. The diff command for every agent is `git diff <baseline HEAD>` plus the untracked files reported by `git status --short` that are not pre-existing.

2. **Agent 0: optimize the prompt**
   - Run this step only in prompt mode and only if the `gimme-prompt` skill is available (listed among the available skills, or `~/.claude/skills/gimme-prompt/SKILL.md` exists). Never run it in plan mode. Otherwise skip it.
   - Run a `general-purpose` subagent (foreground) with: the task description verbatim, the absolute path of `optimized-prompt.md`, and this instruction: "Invoke the `gimme-prompt` skill with the task description as input. You cannot ask the user questions: where the skill says to ask questions, state your assumptions instead. Write to the output path the optimized prompt plus a short list of assumptions, suggested improvements, and detected risks. Do not modify source code."
   - If `optimized-prompt.md` is missing, rerun once, then continue with the original task description only and tell the user in the final step.

3. **Build the shared context**
   - Build a **shared context** block containing: the user's instructions verbatim, the absolute plan path (plan mode) or the contents of `optimized-prompt.md` when it exists (prompt mode; the original task stays as the source of truth if they conflict), every working directory and additional directory of this session, the baseline per repository, the diff command, the pre-existing changes, and the user rules (the full contents of `~/.claude/CLAUDE.md`, plus any rule the user stated in this conversation). Subagents do not see this conversation, so they only know what you pass them.

4. **Agent A: implement**
   - Run the `plan-implementer` subagent (foreground) with the shared context and the absolute path of `implementation.md`.
   - Check that `implementation.md` exists. If it is missing, rerun once, then stop and report the failure.
   - If the diff is empty, stop and report that nothing was implemented, with the reason from `implementation.md`.

5. **Agent B: review**
   - Run the `implementation-reviewer` subagent (foreground) with the shared context and the absolute paths of `implementation.md` and `implementation-review.md`.
   - Check that `implementation-review.md` exists. If it is missing, rerun once, then stop and report the failure.

6. **Agent C: decide and apply**
   - Run the `implementation-arbiter` subagent (foreground) with the shared context and the absolute paths of `implementation.md`, `implementation-review.md`, and `applied.md`.
   - Check that `applied.md` exists. If it is missing, rerun once, then stop and report the failure.

7. **Present**
   - Read `implementation.md` and `applied.md` and show the user:
     - a short summary of what was implemented,
     - **Pending findings**: the review findings that were not applied and need the user's decision, each with its severity, the problem, and the proposed fix,
     - **Decisions worth knowing**: the notable decisions taken by Agent A (deviations from the plan, assumptions, choices the plan left open) and by Agent C (findings rejected or applied differently than proposed),
     - the verification result (typecheck, tests, analyzer),
     - `git status --short` of each affected repository.
   - Link the plan (or `optimized-prompt.md` in prompt mode) and the files so the user can inspect the intermediate steps.
   - Do not apply the pending findings. Do not commit or push. Wait for the user.

## Launching a subagent

- Launch each subagent with the Agent tool, using its `subagent_type` (`general-purpose` for the prompt optimizer, then `plan-implementer`, `implementation-reviewer`, `implementation-arbiter`). The model comes from the frontmatter in `~/.claude/agents/<name>.md`.
- If that `subagent_type` is not available, launch a `general-purpose` subagent and put the body of `~/.claude/agents/<name>.md` at the top of the prompt, followed by the step inputs.

## Constraints

- Run the subagents one at a time. Each step needs the previous file and the code left by the previous agent.
- Do not edit the plan, `optimized-prompt.md`, `implementation.md`, `implementation-review.md`, or `applied.md` yourself.
- Do not modify source code yourself.
- Do not commit, push, stash, reset, or switch branches.
