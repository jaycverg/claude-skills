---
name: review-changes
description: "Review a code changeset OR a plan/spec document. For code: auto-detects the framework(s) in the diff (Java, Next.js/TypeScript, SvelteKit, SQL/database migrations & Liquibase) and reviews against SOLID, DRY, KISS, clean code, architecture, and quality. For a plan/spec doc: reviews for completeness, correctness, feasibility, risk, and scope. Produces a structured findings report and optionally applies approved fixes. Trigger when the user asks to review changes / a diff / a commit / a branch / staged changes, or to review a plan / spec / design doc, or runs /review-changes."
---

# Review Changes

Review either a **code changeset** (detecting the framework(s) and applying the matching criteria) or a **plan/spec document** (reviewing the plan itself for completeness, correctness, feasibility, risk, and scope). Optionally apply approved fixes afterward — and optionally loop (re-review and re-fix until clean).

## Arguments

`$ARGUMENTS` — all optional:

- **Target** — what to review:
  - **Code scope** — a file path, commit hash, commit range, branch name, `--staged`, or `--full`. Resolved per the scope table in `references/common.md`. Defaults to all uncommitted changes.
  - **Plan/spec document** — a path to a plan or spec markdown file (e.g. under `.local/plans/`, `.local/docs/`, a design doc or RFC), or `--plan` / `--spec` to force document mode.
- **`--fix`** — after presenting the report, offer to apply fixes (Phases 4–5). Without it, stop after the report. Treat a natural-language "and fix it" / "review and fix" the same as `--fix`.
- **`--loop`** — run a **review loop**: after each fix pass, re-review and re-fix the same scope until clean (or a stop condition is hit — see Phase 6). Implies `--fix`. When `--fix` is in effect but `--loop` is not, Phase 3 still offers the loop as a choice. Treat "keep fixing until clean" / "iterate until no issues" as `--loop`.

## Phase 0: Determine Review Mode

Decide whether this is a **code review** or a **plan/spec review**:

- **Plan/spec mode** when: `--plan` or `--spec` is given; the user says "review this plan / spec / design doc"; or the target is a single markdown document that reads as a plan/spec (under `.local/plans/` or `.local/docs/`, a filename or heading indicating a plan/spec/RFC/design, prose-and-steps rather than code). When the target is a `.md` file, glance at its content to confirm it's a plan/spec and not, say, documentation being edited as code.
- **Code mode** otherwise (the default).

If the resolved scope (including the no-arg default) contains **only** a single plan/spec document, treat it as plan/spec mode and review the current document — not its diff-as-code. A mixed changeset of code plus a plan doc stays in code mode (review the code; the plan doc falls under "Skip"). If genuinely ambiguous, ask the user which they want.

- **Plan/spec mode** → skip to **Phase 1P** below.
- **Code mode** → continue with Phase 1 (code).

## Phase 1: Scope & Classify (code mode)

1. Resolve the scope from `$ARGUMENTS` per the scope table in `references/common.md`, and gather:
   - Changed files: `git diff --name-only <scope>` (or `git show --name-only <hash>`; for `--full`, all tracked source files).
   - Context: `git status --porcelain` and `git log --oneline -5`.
2. Classify each changed file into a framework bucket:
   - **SQL/Database** — `.sql`; **and** Liquibase changelogs in any format — an `.xml`/`.yaml`/`.yml`/`.json` file containing `databaseChangeLog`, or living under a `changelogs/` or `liquibase` path
   - **Java** — `.java`, `.xml` (pom.xml, Spring configs), `.properties`, `.yml`/`.yaml`
   - **Next.js/TypeScript** — `.ts`, `.tsx`, `.js`, `.jsx`, `.css`, `.json`
   - **SvelteKit** — `.svelte`, `.ts`, `.js`, `.css`, `.json`
   - **Skip** — anything else (images, lockfiles, generated files)
   - **Disambiguate TS/JS** by repo signals, not extension alone: `.svelte` files or `svelte.config.*` → SvelteKit; `next.config.*` or an `app/`+`pages/` layout → Next.js. If still ambiguous, ask the user.
   - **Disambiguate XML/YAML/JSON**: route to **SQL/Database** when the file is a Liquibase changelog (contains `databaseChangeLog`, or sits under `changelogs/`/`liquibase`) — otherwise an `.xml`/`.yml` config or `pom.xml` stays **Java**, and a `.json`/`.css` stays with its JS framework.
3. If no reviewable changes exist in the scope, tell the user and stop.

## Phase 2: Review Per Framework

For each framework bucket present, load its criteria reference and review the files in that bucket:

- SQL/Database → `references/sql.md`
- Java → `references/java.md`
- Next.js/TypeScript → `references/nextjs.md`
- SvelteKit → `references/svelte.md`

For each changed file: read the full diff and surrounding code for context, then evaluate against the criteria in the matching reference. Report only actionable findings — skip categories where the change is clean.

If the changeset spans **multiple** frameworks, spawn one subagent per framework (Agent tool) so reviews run in parallel. Give each subagent: the file list for its bucket, and — **inlined into the prompt** — the full contents of its criteria reference plus the report format from `references/common.md`. Read those reference files yourself and paste their text in; do **not** hand the subagent a file path. (A subagent runs from the project root and doesn't know where this skill is installed — the config home varies by `CLAUDE_CONFIG_DIR` — so a skill-relative or hardcoded install path won't reliably resolve for it. You, the skill's own agent, can resolve `references/*.md` and inline them.)

After reviewing, go to **Phase 3: Report**.

## Phase 1P: Load the Plan (plan/spec mode)

1. Read the **entire** target document. If a scope selector resolved to a diff of the document (e.g. reviewing an edited plan), still read the full current document for context — review the plan as it now stands.
2. Gather the context the plan depends on: read the code, configs, and files the plan references so you can check its assumptions against reality. The most valuable plan findings come from catching where the plan misstates how the current system behaves.
3. If the document is empty or isn't actually a plan/spec, tell the user and stop.

## Phase 2P: Review the Plan

Evaluate the document against the criteria in `references/plan.md`. Cover completeness, correctness/feasibility, architecture fit, risk/edge cases, scope/sequencing, and clarity. Report only actionable findings — skip categories where the plan is solid.

Then go to **Phase 3: Report**.

## Phase 3: Report

Consolidate all findings into a single report using the format, severity levels, and rules in `references/common.md`. Deduplicate issues that recur across files or frameworks (code mode) or across sections (plan mode). In **plan/spec mode**, apply the report adaptations in `references/plan.md` (plan-specific principle labels, section-based `File` citations, plan severity definitions, and an Open Questions subsection where relevant).

**If fixing** (`--fix` or the user asked to fix): add a **Fixable:** `yes | no` line to each finding (`yes` = auto-fixable, `no` = needs human judgment), then **stop and ask**:

> **Which findings should I fix?**
> - `all` — fix everything marked Fixable: yes
> - `critical` — fix only critical items
> - `1,3,5` — fix specific findings by number
> - `none` — done, just wanted the review
>
> **Iterate?** Reply `loop` (alone or with a selection, e.g. `all loop`) to re-review and re-fix until no findings remain in the selected scope — otherwise I'll do a single fix pass.

If `--loop` was passed (or the user said "iterate until clean" / "keep fixing until clean"), skip this iterate question — looping is already chosen; still confirm the **selection** (which findings) unless that's also unambiguous.

Record the **fix selection** (`all` / `critical` / a number list) and whether **looping** is on — both carry through every iteration of the loop without re-asking.

**Numeric selections and looping:** a number list (`1,3,5`) identifies findings in the *current* report only — a re-review renumbers everything, so the list can't be replayed. If a numeric selection is combined with looping, apply it literally on the first pass, then for the remaining iterations fall back to the broadest severity class among the chosen findings (e.g. a Critical + a Warning were picked → continue looping on **all** fixable findings; only Criticals were picked → continue as `critical`). State this transition when it happens.

**If not fixing:** stop here — the report is the deliverable.

## Phase 4: Fix (only when requested and approved)

**Code mode** — for each approved finding:
1. Read the full file (not just the diff) for complete context.
2. Apply the fix with the Edit tool.
3. After all fixes in a file, run the project's lint/format if available:
   - Java: `mvn spotless:apply` (or the project formatter)
   - Next.js: `pnpm run lint --fix && pnpm run format`
   - SvelteKit: `npm run lint -- --fix && npm run format`
   - SQL/Database: no formatter step — instead sanity-check the edited statement parses (e.g. Liquibase `validate`/`status` if the project is set up for it) and re-confirm any index/charset assumptions against the live schema. **Never** run the migration against a real database as part of "fixing."

If a fix breaks lint or introduces a new issue, revert that specific fix and report it.

**Plan/spec mode** — for each approved finding, edit the plan document with the Edit tool to address it: revise the affected step/section, fill gaps, resolve ambiguities, or add missing concerns (testing, rollback, edge cases). Keep the document's structure and tone; don't rewrite wholesale. For Open Questions you can't resolve yourself, surface the decision to the user rather than guessing — don't silently pick an answer.

## Phase 5: Verify & Summarize

1. Confirm nothing broke:
   - **Code mode** — run the project's build or type-check:
     - Java: `mvn compile -q`
     - Next.js: `pnpm run build` or `npx tsc --noEmit`
     - SvelteKit: `npm run check` or `npm run build`
     - SQL/Database: no build/compile. Verify by re-reading the statements for syntax and by re-checking index/charset/type assumptions against `information_schema` (read-only); where useful, attach an `EXPLAIN` of any changed migration query rather than executing it. Do not run DML/DDL to "verify."
   - **Plan/spec mode** — re-read the edited document end-to-end to confirm it's internally consistent, the edits resolve the approved findings, and no stale/contradictory text was left behind. (No build step.)
2. Show a summary:
   ```
   ## Fix Summary

   **Applied:** X fixes
   **Skipped:** Y (not approved or not auto-fixable)
   **Build:** passing | failing (details)   ← code mode only; omit for plan/spec mode

   ### Changes Made
   | # | File | Finding | Status |
   |---|------|---------|--------|
   | 1 | path/to/file:line | [Critical] Title | Fixed |
   | 5 | path/to/file:line | [Warning] Title | Skipped — lint failed |
   ```
3. Show the diff: `git diff <path>` (code mode, or a plan doc that's tracked). If the plan doc is git-ignored (e.g. under `.local/`), `git diff` won't show it — show the relevant edited sections instead.

**If looping is on**, don't show the per-pass diff each round — go to Phase 6. Save the full diff/summary for the final pass.

## Phase 6: Loop (only when looping was chosen)

Re-run the review on the **same scope** and re-apply fixes using the **same fix selection** the user already chose — without re-asking. Repeat until a stop condition is met.

The recorded fix selection defines the loop's **target set** — the findings it's responsible for resolving (`all` = every `Fixable: yes` finding; `critical` = critical findings; a numeric list → resolved to a severity class per the Phase 3 rule above).

Each iteration:
1. Re-review the scope (Phase 1/1P → 2/2P → 3). In the loop, **don't** re-prompt for the fix selection — reuse the recorded one.
2. If findings in the target set are marked `Fixable: yes`, fix them (Phase 4) and verify (Phase 5, step 1).
3. Emit a one-line progress note per iteration: `Iteration N: <found> findings → <fixed> fixed, <remaining> remaining`.

**Stop the loop when any holds:**
- **Clean** — no findings remain in the target set. (Success.)
- **No progress** — an iteration applies zero fixes, or the same finding recurs unchanged across two consecutive iterations (a fix that can't converge — avoids an infinite loop).
- **Verification failed** — the build/type-check (code) breaks, or the document became inconsistent (plan); stop and report rather than fixing atop a broken state.
- **Cap reached** — a maximum of **5 iterations**. If still not clean, stop and report what remains.
- **Only non-fixable findings remain** — everything left is `Fixable: no` (needs human judgment); stop and list them for the user.

After the loop ends, produce **one** final consolidated report:
- The reason the loop stopped (clean / cap / no-progress / verification failed / only human-judgment items left).
- A combined Fix Summary (Phase 5 format) across all iterations, plus the iteration count.
- Any remaining findings the loop did not resolve, with why.
- The final diff (or edited sections, per Phase 5 step 3).

## Resources

### references/
- `common.md` — scope selectors + report format, severity levels, and rules (shared by code and plan modes)
- `sql.md` — SQL / database script & Liquibase migration review criteria (indexing, backfills, locking, charset, prod safety)
- `java.md` — Java / Spring review criteria
- `nextjs.md` — Next.js / TypeScript / React review criteria
- `svelte.md` — SvelteKit / TypeScript review criteria
- `plan.md` — plan / spec document review criteria (completeness, correctness, feasibility, risk, scope)
