---
allowed-tools: Read, Write, Edit, Bash, Grep, Glob, Agent, AskUserQuestion
argument-hint: <implementation prompt> [--devs N] [--reviewers N] [--manual-approval]
description: Implement a feature or plan as an agent team — by default 1 dev implements in an isolated worktree, 1 reviewer tests and approves. Auto-merges PRs when the project's CLAUDE.md specifies a PR workflow.
---

# Implement

Implement the following as an agent team: $ARGUMENTS

This command **always** uses the agent team feature. It **orchestrates** the work — it does not implement the change itself. All edits happen inside the dev subagent(s) so role separation between implementer and reviewer is real.

## Phase 1: Parse arguments

Extract from `$ARGUMENTS`:
- **Prompt** — the implementation task (everything that isn't a flag). May be an inline description or a path to a plan file (e.g. under `.local/plans/`). If it's a plan file, the dev should read it in full.
- **`--devs N`** (optional, default `1`) — number of developer agents.
- **`--reviewers N`** (optional, default `1`) — number of reviewer agents.
- **`--manual-approval`** (optional) — never auto-merge; the human approves and merges, even if the project specifies a PR workflow.

If `--devs > 1`, the work must split cleanly into independent sub-tasks. Propose the split via `AskUserQuestion` and wait for confirmation before spawning. If a clean split isn't possible, fall back to 1 dev and tell the user why.

When proposing the split, also identify the **shared surfaces** the sub-tasks touch in common — validation/contract rules, shared helpers, navigation/persistence utilities, shared layout/CSS, common test helpers. Assign each shared surface a single owner and flag the rest as explicit coordination points. A contract change by one dev (e.g. adding step-validation) routinely regresses a sibling's work and the shared test helpers — surface these up front instead of discovering them at integration time.

## Phase 2: Detect the project workflow

Read the project's root `CLAUDE.md` (and any nested `*/CLAUDE.md` that applies to the working directory). Look for an explicit team / PR workflow. Strong signals:

- Phrases like "pull request", "open a PR", "merge via PR", "feature branch", "branch off", "PR-vs-target-branch".
- An explicit "Agent Team Mode" (or similar) section describing branch naming, PR creation, reviewer ping, merge rules.
- Project-specific conventions: branch prefixes (e.g. `<jira-ticket>-*`), a custom worktree layout, Jira transitions, reviewer-merge size limits.

Decide `workflowMode`:

- **`pr-auto-merge`** — `CLAUDE.md` describes a PR workflow **and** `--manual-approval` was **not** passed. Reviewer merges the PR after approval.
- **`pr-manual-merge`** — `CLAUDE.md` describes a PR workflow **but** `--manual-approval` was passed (or the project's own rules say the human merges in this case — e.g. a "simple fix" vs "risky change" rule). Reviewer approves; the human merges.
- **`ask`** — no workflow is described anywhere. Ask the user via `AskUserQuestion`:

  > The project doesn't specify a team workflow. How should the dev and reviewer collaborate?
  >
  > - **Direct on current branch** — dev edits the working tree (still in its own worktree); reviewer reads the diff and approves; no PR.
  > - **Feature branch + manual merge** — dev creates a feature branch; reviewer approves; the human merges.
  > - **Feature branch + PR + auto-merge** — dev creates a feature branch and opens a PR; reviewer approves and merges.

  Record the answer as the resolved `workflowMode`.

### Confirm the base branch (always)

Regardless of `workflowMode`, **always** confirm which branch the work should branch off before spawning anything. Do not silently default to `master` or the current branch.

1. Determine candidate bases: the current branch (`git rev-parse --abbrev-ref HEAD`), the project's main branch, and any in-flight dep tip the project conventions point to (e.g. a "branch off latest fixed dep tip, not master" rule).
2. Ask the user via `AskUserQuestion` which base to branch off, offering those candidates. Put the project-convention-recommended base first.
3. Record the confirmed base branch and pass it to the dev(s) as the explicit branch-off point.

State the resolved `workflowMode` **and** the confirmed base branch to the user before spawning anything, so they can correct either.

## Phase 3: Spawn the dev agent(s)

For each developer (default 1), spawn an Agent subagent. Give each a unique `name` (e.g. `dev-1`) so you can resume it via `SendMessage` later. Brief each dev with:

> You are the **dev** on an agent team of `<devs>` developer(s) + `<reviewers>` reviewer(s).
>
> **Task:** `<prompt>` *(or this dev's sub-slice if multi-dev; if the task references a plan file, read it in full first)*
> **Workflow mode:** `<workflowMode>`
> **Base branch (confirmed):** `<baseBranch>`
>
> Follow this workflow exactly:
> 0. **Worktree (always)** — do all your work in a **new, separate git worktree**, never in the shared checkout. Create it from the confirmed base branch: `git worktree add <path> <baseBranch>` (use the project's worktree layout if `CLAUDE.md` specifies one; otherwise default to `<repo-root>/.claude/worktrees/<your-name>/`). Ensure `.claude/worktrees/` is git-ignored (check `.gitignore`, `.git/info/exclude`, and the global excludesfile); if it isn't, note it in your report. All branch creation, edits, commits, tests, and typecheck happen inside that worktree.
> 1. **Branch setup** — branch off the confirmed base `<baseBranch>` (do not pick your own base). For `pr-*` modes, create a feature branch following project naming conventions. Do **not** set a remote tracking branch (no `--track`). For `direct-on-current-branch`, still work in the new worktree checked out from `<baseBranch>`.
> 2. **Explore** — read relevant code, understand existing patterns, identify files to change.
> 3. **Plan** — briefly outline the approach.
> 4. **Execute** — make changes following existing style and conventions. No unrelated improvements. Adhere to SOLID, DRY, KISS, and clean code principles.
> 5. **Verify** — a green build/typecheck does **not** prove the artifact is correct. Verify the actual runtime/rendered behavior of the change (run it, or exercise it via an integration/e2e check). Some toolchains silently ship broken output on a "successful" build — CSS that compiles but never loads, an asset that 404s, a bundle that doesn't hydrate — so confirm the real output, not just the exit code.
> 6. **Tests** — detect the project's test setup. If the project has tests: update affected tests, remove tests no longer valid, and add coverage for new behavior; ensure the suite passes. If the project has **no** tests, note that in your report so the orchestrator can ask the human whether to add a scaffold — do not add one unprompted. Each test must assert the **specific expected state** — never a disjunction that also accepts the broken/degraded/empty outcome (e.g. "renders results OR shows the empty state"), which passes green while the feature is broken. If a real assertion is blocked on an unbuilt dependency, mark it `test.fixme` with a one-line note rather than weakening it to pass.
>
> Honor any project-specific team-mode instructions in `CLAUDE.md`: Jira transitions, project-specific worktree layout, branch prefixes, PR-gate scripts (e.g. `npm run prepare-pr`), worklog comment format, runtime version managers (`.nvmrc`/`.sdkmanrc`), `.env` loading, etc.
>
> When done, report back:
> 1. Worktree path
> 2. Branch name (or "current branch")
> 3. Commit SHA(s)
> 4. Files changed
> 5. Test / build / typecheck results (and whether the project had tests at all)
> 6. A 2-3 sentence summary of the change and any noteworthy decisions
>
> Do **not** push or open a PR — that's the reviewer's job (in `pr-*` modes).

For multi-dev, send all spawn calls in a single message so they run in parallel. Wait for every dev to complete before Phase 4.

## Phase 4: Spawn the reviewer agent(s)

Spawn a reviewer subagent for each reviewer (default 1). Brief with:

> You are the **reviewer** on this agent team. The dev(s) just completed work:
>
> - Worktree(s): `<paths>`
> - Branch(es): `<branches>`
> - Commit SHA(s): `<shas>`
> - Files changed: `<list>`
> - Dev summary: `<summary>`
> - Workflow mode: `<workflowMode>`
>
> Your job:
> 1. **Read the diff** — `git log <base>..<branch>` and `git diff <base>..<branch>` (or `git diff` for direct-on-current-branch).
> 2. **Review** — apply the project's review criteria. If the change is Java, Next.js, or SvelteKit, use the `/review-changes` skill (it auto-detects the stack). Otherwise evaluate against SOLID, DRY, KISS, clean code, architecture, and security.
> 3. **Test** — run the project's test suite, build, typecheck, and lint. Run the PR-gate script if the project has one (e.g. `npm run prepare-pr`). The work is **not approved** unless these pass.
> 4. **Verify UI/visual changes against the design source** (when the change is user-facing) — compare computed styles / screenshots to the referenced design tokens, **not** a DOM/`innerHTML` check (DOM checks can't see CSS pseudo-elements or computed values). Verify on the **actual running build with caches defeated** (block the service worker / hard-refresh) so a stale cache can't mask the change or fake a pass.
> 5. **Decide** — `approve` or `request-changes`, with concrete reasoning and file/line references for any findings.
>
> If `workflowMode` is `pr-auto-merge` or `pr-manual-merge` and the branch hasn't been pushed yet, push it (`git push -u origin <branch>`) and open the PR with `gh pr create`. Use the project's PR title/body conventions (Jira key in title, "## Summary" + "## Test plan" body, etc.) if `CLAUDE.md` specifies them.
>
> Report back with: verdict (`approve` | `request-changes`), findings, gate/test/build/lint results, and PR URL if you opened one.

## Phase 4½: Integration gate (before approval)

When the change spans **multiple files / steps / devs** *or* touches a **user-facing flow**, run **one end-to-end happy-path** before approval — fill real data → submit/complete → assert the real outcome — not only per-file unit tests. This is the gate that catches seam bugs: an orphaned cross-step fetch, an empty payload reaching the API, a contract change regressing a sibling. For multi-dev runs, run it against the **merged** integration branch after all devs merge, before reviewers sign off. A change that is genuinely single-file and not user-facing can skip this gate — say so explicitly rather than silently.

## Phase 5: Handle the verdict

**If any reviewer requests changes:**
1. Consolidate findings across reviewers (dedupe overlapping issues).
2. Relay them to the responsible dev — prefer `SendMessage` to the running dev agent (preserves its context); fall back to re-spawning with the original task + the new findings if the agent isn't reachable.
3. Wait for the dev to apply fixes and report a new SHA.
4. Loop back to Phase 4 with the updated state. **Cap at 3 review rounds** — after that, stop and surface the impasse to the human with a summary of what's been tried.

**If all reviewers approve:**

- **`pr-auto-merge`** — the reviewer (or this command) merges the PR. Use the project's preferred strategy if `CLAUDE.md` says so (`--squash`, `--merge`, `--rebase`); default to `gh pr merge --squash --delete-branch`. After merging, run any project post-merge steps mentioned in `CLAUDE.md` (e.g. transition the Jira ticket to Done).
- **`pr-manual-merge`** — stop. Surface the PR URL and approval summary to the human; do **not** merge.
- **`direct-on-current-branch`** — stop. The work is on the dev's worktree branch; surface the diff summary and worktree path to the human.

### If the project had no tests

If the dev reported that the project has no tests and `workflowMode` is not blocking on it, ask the human: "This project has no tests. Do you want to add tests for the changes?" If yes, send the dev back to add a minimal test scaffold appropriate for the stack plus coverage for the new behavior, then re-review. If no, proceed.

## Phase 6: Final report

One short summary to the user:
- Resolved `workflowMode` and confirmed base branch
- Worktree path(s) created (and a reminder to `git worktree remove` them once merged)
- Branch(es) used and final commit SHA(s)
- PR URL(s) (if any) and merge status
- Review rounds taken
- Any project-specific post-merge actions performed (or pending, if manual)

## Notes

- Subagents start with a fresh context — pass the full task prompt in the brief, don't assume they can see this conversation.
- Every dev works in its **own new worktree** (one per dev, never shared), branched off the **user-confirmed base** — never the shared checkout and never an unconfirmed default. Report each worktree path so the human can clean them up (`git worktree remove <path>`) once merged.
- This command **orchestrates**; it does not implement the work itself. All edits happen inside the dev subagents so role separation is real.
- If the project's `CLAUDE.md` contradicts a default in this command (branch naming, base branch, merge strategy, worktree layout, post-merge hooks), the project wins.
