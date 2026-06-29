---
allowed-tools: Read, Grep, Glob, Bash, Edit, Write, Agent, AskUserQuestion, Skill, ExitPlanMode
argument-hint: <prompt or path to spec file>
description: Create a plan from a prompt or spec file, then hand it to review-changes to fix and loop until the plan is clean.
---

# Create Plan

Create a plan based on the following input: $ARGUMENTS

## Rules

- **Switch to plan mode immediately.** Do not write or modify any file during planning.
- **Apply KISS, SOLID, DRY, and clean code principles** when shaping the plan.
- **Do not speculate** about requirements not present in the prompt or codebase.
- **Do not add features, abstractions, or error handling** beyond what the prompt requires.
- **Keep steps concrete and scoped.** Avoid vague guidance like "handle edge cases" — name the specific edge cases, inputs, or code paths.
- **Do not write any file during the planning phases (1–3).** The plan is only persisted in Phase 4, after the user accepts it.
- **Own the seams.** When the work decomposes into multiple pieces, assign every cross-piece data dependency (fetch, shared state, persistence/sync, contract) to a step. Logic that spans pieces but is owned by none is the most common gap.
- **Define how "done" is verified**, including **one end-to-end happy-path acceptance**. Acceptance must assert the specific correct state — a degraded fallback must not count as success.
- **Do not implement.** The deliverable is a reviewed plan, not code changes.

## Phase 1: Understand the Input

1. Determine whether `$ARGUMENTS` is a direct prompt or a path to a spec file.
   - If it looks like an existing file path, read the file with `Read`.
   - Otherwise, treat it as the prompt text itself.
2. Identify the concrete deliverable. If the requirement is ambiguous, ask the user a targeted clarifying question via `AskUserQuestion` before planning. Do not invent requirements to fill gaps.

## Phase 2: Explore the Codebase

1. Read the relevant files and understand existing patterns, conventions, and structure.
2. For broad exploration use the `Agent` tool with `subagent_type=Explore`. For targeted lookups use `Grep` / `Glob` / `Read` directly.
3. Only gather context that is needed to plan the specific change. Stop exploring once you have enough to produce a concrete plan.
4. **If the prompt/ticket references a design (Figma node, mockup image), capture the exact design tokens** for the in-scope elements — font family / size / weight / line-height, colors as hex, spacing, dimensions — and carry them into the Verification section as explicit acceptance values. "Match the mockup" is not a criterion; the tokens are. Note the verification method: pull the full-resolution render via the Figma REST API (a separate quota from the rate-limited MCP) and diff it against computed styles.

## Phase 3: Present the Plan

Present the plan clearly using this structure:

1. **Goal** — one or two sentences restating what will be built, grounded in the prompt.
2. **Scope** — what is in scope and what is explicitly out of scope.
3. **Files to change** — list each file with a brief note on what changes there (new file, modified function, etc.).
4. **Steps** — ordered, concrete steps. Each step should name the file, function, or symbol it touches and describe the change in specific terms.
5. **Seams & cross-cutting concerns** — *required whenever the change spans more than one step / component / ticket.* List the data and behavior that flow **across** the pieces — fetches, shared state, persistence/sync, validation contracts, shared layout/CSS — and assign **each one an owning step**. Every cross-piece data dependency must have an explicit owner: decomposing by UI piece tends to orphan the fetch or sync that lives in the seam, which is exactly how a question-survey fetch or a `KEY_POST_DATA` sync ends up built by no step. If the change is single-piece, write "N/A — single component."
6. **Verification & Acceptance** — for each requirement, how it is verified. **Mandatory:** at least one **end-to-end happy-path acceptance** that exercises the whole flow the change participates in (fill real data → act → assert the real outcome), not only per-unit/per-step checks. Acceptance must assert the **specific correct end state** — never a disjunction that also passes on the degraded/empty/error state ("renders results OR the empty state" is not acceptance).
7. **Assumptions** — any assumptions made. If there are none, say so. Do not invent assumptions to pad the plan.
8. **Open questions** — anything the user must decide before implementation. If none, say so.

Then call `ExitPlanMode` to hand the plan over for the user's review. Do not proceed to implementation. Once the user accepts the plan, continue to Phase 4.

## Phase 4: Save & Hand Off to review-changes

After the plan is accepted, persist it and — with the user's confirmation — chain it into the `review-changes` skill, which reviews the plan document, applies fixes, and loops until it's clean.

1. **Save the plan.** Write the full plan verbatim to `.local/plans/<YYYY-MM-DD>-<short-kebab-slug>.md` (create the folder if needed; e.g. `2026-05-28-add-rate-limit-headers.md`). Use today's date and a slug derived from the goal.
2. **Ask whether to run review-changes.** Use `AskUserQuestion` to confirm whether to hand the saved plan off to `review-changes` now. Offer a "Yes, review now" option (recommended) and a "No, stop here" option.
   - If the user declines, skip to step 4 (report the saved path and stop — do not invoke review-changes).
3. **Hand off.** Only if the user confirmed: invoke the `review-changes` skill via the `Skill` tool with the saved path and `--plan --fix --loop`, e.g. `review-changes .local/plans/<file>.md --plan --fix --loop`. This runs review-changes in **plan/spec mode** against the document and loops — re-reviewing and re-fixing the same plan until no findings remain (or a stop condition in the skill's Phase 6 is hit: cap reached, no progress, or only human-judgment items left).
   - Let review-changes drive its own fix selection (`all`) and loop; don't re-prompt for confirmation here — the loop is the point of the chain.
   - This plan-mode review specifically confirms the **Seams & cross-cutting concerns** and **Verification & Acceptance** sections are present and non-trivial (not "N/A" where the change clearly spans pieces, and with a real end-to-end acceptance, not a disjunction).
   - For any finding review-changes flags as `Fixable: no` (needs a human decision), surface it to the user rather than guessing.
4. **Report and stop.** Report the final saved plan path and, if review ran, the review outcome (clean / what remains). Do **not** implement the plan — the user will run their own `/implement <plan-file>` when ready.
