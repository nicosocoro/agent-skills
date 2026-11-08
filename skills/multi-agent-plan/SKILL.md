---
name: multi-agent-plan
description: Build an implementation plan with three subagents (planner, plan-reviewer, plan-arbiter) without user intervention. Use when the user invokes /multi-agent-plan, says "multi-agent-plan", or asks for a multi-agent plan.
disable-model-invocation: true
---

# multi-agent-plan

Orchestrate three subagents in sequence. You are the orchestrator: you do not plan, review, or decide yourself, and you do not ask the user anything until the final step.

## Workflow

1. **Prepare**
   - Take the task description from the user's message (`$ARGUMENTS`). If no task was given, ask for it and stop.
   - Pick a short kebab-case slug for the task.
   - Create the folder `<primary workspace root>/.claude/plans/<slug>/`. The primary workspace root is the repository the task mainly affects, or the current working directory when unclear.
   - Paths used below: `plan.md`, `review.md`, `final-plan.md` inside that folder.
   - Build a **shared context** block containing: the full task description verbatim, every working directory and additional directory of this session, the user rules (the full contents of `~/.claude/CLAUDE.md`, plus any rule the user stated in this conversation), and any file the user attached or mentioned. Subagents do not see this conversation, so they only know what you pass them.

2. **Agent A: plan**
   - Run the `planner` subagent (foreground) with the shared context and the absolute path of `plan.md`.
   - Check that `plan.md` exists and is not empty. If it is missing, rerun the planner once, then stop and report the failure.

3. **Agent B: review**
   - Run the `plan-reviewer` subagent (foreground) with the shared context, the absolute path of `plan.md`, and the absolute path of `review.md`.
   - Check that `review.md` exists. If it is missing, rerun the reviewer once, then stop and report the failure.

4. **Agent C: decide**
   - Run the `plan-arbiter` subagent (foreground) with the shared context and the absolute paths of `plan.md`, `review.md`, and `final-plan.md`.
   - Check that `final-plan.md` exists. If it is missing, rerun the arbiter once, then stop and report the failure.

5. **Present**
   - Read `final-plan.md` and show the user: a short summary of the plan, the Decisions table, and the open questions.
   - Link the three files so the user can inspect the intermediate steps.
   - Do not implement the plan. Wait for the user.

## Launching a subagent

- Launch each subagent with the Agent tool, using its `subagent_type` (`planner`, `plan-reviewer`, `plan-arbiter`). The model comes from the frontmatter in `~/.claude/agents/<name>.md`.
- If that `subagent_type` is not available, launch a `general-purpose` subagent and put the body of `~/.claude/agents/<name>.md` at the top of the prompt, followed by the step inputs.

## Constraints

- Run the subagents one at a time. Each step needs the previous file.
- Do not edit `plan.md`, `review.md`, or `final-plan.md` yourself.
- Do not modify source code during this workflow.
