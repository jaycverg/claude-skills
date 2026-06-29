# claude-skills

A small collection of [Claude Code](https://docs.claude.com/en/docs/claude-code) commands and skills for a plan → implement → review workflow.

## Contents

| Path | Type | What it does |
| --- | --- | --- |
| `commands/create-plan.md` | Slash command | Create a plan from a prompt or spec file, then hand it to `review-changes` to fix and loop until the plan is clean. |
| `commands/implement.md` | Slash command | Implement a feature or plan as an agent team — by default 1 dev implements in an isolated worktree, 1 reviewer tests and approves. Auto-merges PRs when the project's CLAUDE.md specifies a PR workflow. |
| `skills/review-changes/` | Skill | Review a code changeset **or** a plan/spec doc. For code it auto-detects frameworks in the diff (Java, Next.js/TypeScript, SvelteKit, SQL/Liquibase) and reviews against SOLID, DRY, KISS, clean code, architecture, and quality. Produces a structured findings report and optionally applies approved fixes. |

The `review-changes` skill loads per-framework guidance on demand from `skills/review-changes/references/` (`common`, `java`, `nextjs`, `svelte`, `sql`, `plan`).

## How it fits together

```
/create-plan  ──►  writes a plan, loops it through ──►  review-changes (plan mode)
/implement    ──►  dev + reviewer agents, reviewer uses ──►  review-changes (code mode)
```

`review-changes` is the shared reviewer used by both commands, and can also be invoked directly.

## Installation

Drop the files into your Claude Code config directory (`~/.claude` for user-level, or a project's `.claude`):

```sh
# user-level install
cp commands/*.md            ~/.claude/commands/
cp -r skills/review-changes ~/.claude/skills/
```

Commands then run as `/create-plan` and `/implement`; the skill triggers on review requests or `/review-changes`.

## Notes

These are tuned to my own stacks (Java backend; TypeScript/React/Next.js/SvelteKit frontend) and personal conventions — adapt the reference files and frontmatter to your own setup.
