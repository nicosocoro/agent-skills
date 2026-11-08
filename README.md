# agent-skills

Skills for [Claude Code](https://claude.com/claude-code) that hand a task to a chain of subagents. Each subagent writes its result to a file, and the next one reads that file. You only step in at the end.

## Skills

### `multi-agent-plan`

Builds an implementation plan without asking you anything along the way.

| Step | Subagent | Output |
|------|----------|--------|
| A | `planner` | `plan.md`: explores the code and drafts the plan |
| B | `plan-reviewer` | `review.md`: checks the plan against the real code and lists bugs, compatibility risks, inconsistencies and improvements |
| C | `plan-arbiter` | `final-plan.md`: accepts or rejects each finding and writes the final plan |

The files are saved in `<repo>/.claude/plans/<task-slug>/`. The skill shows you the final plan, the decision table and any open questions. It doesn't implement anything.

```
/multi-agent-plan add rate limiting to the public API
```

### `multi-agent-review`

Reviews the changes on the current branch against `main` (or `master` if there's no `main`). Uncommitted changes are included. It then applies the fixes the subagents agree on.

| Step | Subagent | Output |
|------|----------|--------|
| A | `code-review-planner` | `review.md`: findings with evidence and a fix plan |
| B | `code-review-verifier` | `adjusted-plan.md`: reviews the diff independently, then confirms, adjusts or discards each finding |
| C | `code-review-fixer` | `applied.md`: applies the fixes, runs the tests and linter, and lists what it skipped |

The files are saved in `~/Documents/reviews/<repo>/<branch>/multi-agent/`. The skill never commits, pushes or switches branches.

```
/multi-agent-review
/multi-agent-review path/to/repo
```

Both skills only run when you call them. Claude never starts them on its own.

## Requirements

Each skill launches subagents by name. Those subagents must be defined in `~/.claude/agents/`:

- `multi-agent-plan`: `planner.md`, `plan-reviewer.md`, `plan-arbiter.md`
- `multi-agent-review`: `code-review-planner.md`, `code-review-verifier.md`, `code-review-fixer.md`

Some of these subagents also follow the checklists in the `review-plan` and `review-branch` skills.

The subagents, `review-plan` and `review-branch` are not in this repository yet.

## Installation

Link each skill into your Claude Code skills folder:

```bash
git clone <this-repo-url> ~/repos/agent-skills
cd ~/repos/agent-skills
mkdir -p ~/.claude/skills
for skill in skills/*/; do
  ln -s "$PWD/$skill" ~/.claude/skills/"$(basename "$skill")"
done
```

Start a new Claude Code session to load the skills. They're symlinks, so `git pull` updates them.

## Layout

```
skills/
  multi-agent-plan/SKILL.md
  multi-agent-review/SKILL.md
```
