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

### `review-plan` and `review-branch`

These are the checklists the reviewer subagents follow. You can also run them on their own:

- `/review-plan path/to/plan.md` reviews a plan file and writes the findings next to it.
- `/review-branch` reviews a branch against its base and writes a report to `~/Documents/reviews/<repo>/`.

These skills only run when you call them. Claude never starts them on its own.

## How the pieces connect

The skills in `skills/` run the workflow, and the subagents in `agents/` do the work. Claude Code finds subagents by name in `~/.claude/agents/`, so install both folders.

| Skill | Subagents it launches | Checklist they follow |
|-------|-----------------------|-----------------------|
| `multi-agent-plan` | `planner`, `plan-reviewer`, `plan-arbiter` | `review-plan` |
| `multi-agent-review` | `code-review-planner`, `code-review-verifier`, `code-review-fixer` | `review-branch` |

The subagents also follow your personal rules in `~/.claude/CLAUDE.md` and each repository's own rules (`CLAUDE.md`, `AGENTS.md`, `.cursor/rules/*.mdc`).

## Installation

Claude Code loads skills from `~/.claude/skills/` and subagents from `~/.claude/agents/`. Clone the repo anywhere, then link each skill and subagent into those folders. With links, `git pull` updates them in place.

If a skill or subagent with the same name already exists there, the link command fails. Rename or remove the old one first.

### macOS and Linux

In a terminal:

```bash
git clone <this-repo-url> ~/repos/agent-skills
cd ~/repos/agent-skills
mkdir -p ~/.claude/skills ~/.claude/agents
for skill in skills/*/; do
  ln -s "$PWD/${skill%/}" ~/.claude/skills/"$(basename "$skill")"
done
for agent in agents/*.md; do
  ln -s "$PWD/$agent" ~/.claude/agents/"$(basename "$agent")"
done
```

To check the links:

```bash
ls -l ~/.claude/skills ~/.claude/agents
```

### Windows

The Claude Code folders are `%USERPROFILE%\.claude\skills` and `%USERPROFILE%\.claude\agents`.

The skills are folders, so they use junctions, which work without extra permissions. The subagents are single files, so they need symbolic links. Windows only lets you create those if one of these is true:

- **Developer Mode** is on: Settings → System → For developers → Developer Mode. This is the recommended option.
- PowerShell is running as administrator.

In PowerShell:

```powershell
git clone <this-repo-url> "$HOME\repos\agent-skills"
cd "$HOME\repos\agent-skills"
New-Item -ItemType Directory -Force "$HOME\.claude\skills", "$HOME\.claude\agents" | Out-Null

Get-ChildItem skills -Directory | ForEach-Object {
  New-Item -ItemType Junction -Path "$HOME\.claude\skills\$($_.Name)" -Target $_.FullName
}
Get-ChildItem agents -Filter *.md | ForEach-Object {
  New-Item -ItemType SymbolicLink -Path "$HOME\.claude\agents\$($_.Name)" -Target $_.FullName
}
```

To check the links:

```powershell
Get-ChildItem "$HOME\.claude\skills", "$HOME\.claude\agents" | Select-Object Name, LinkType, Target
```

If you can't turn on Developer Mode or run as administrator, copy the files in `agents/` into `%USERPROFILE%\.claude\agents` instead. Copies don't update with `git pull`, so copy them again after each pull.

### After installing

Start a new Claude Code session, then run `/multi-agent-plan` or `/multi-agent-review`.

## Layout

```
agents/
  planner.md
  plan-reviewer.md
  plan-arbiter.md
  code-review-planner.md
  code-review-verifier.md
  code-review-fixer.md
skills/
  multi-agent-plan/SKILL.md
  multi-agent-review/SKILL.md
  review-branch/SKILL.md
  review-plan/SKILL.md
```
