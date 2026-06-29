# Plan / Spec Review Criteria

Applies when the review target is a **plan or spec document** (e.g. a markdown file under `.local/plans/` or `.local/docs/`, a design doc, an RFC, or a feature spec) rather than a code changeset.

The deliverable is not "is this code clean" but **"will executing this plan produce the right result, and is it the simplest plan that does so."** Read the *entire* document first, then evaluate against the categories below. Where the plan references existing code, read that code to check the plan's assumptions against reality — a plan that misstates how the current system behaves is the most common and most dangerous failure.

Report only actionable findings — skip categories where the plan is solid.

### Completeness

- Are all parts of the stated goal actually covered by the steps? Map each goal/requirement to the step(s) that satisfy it; flag any goal with no corresponding step.
- Missing concerns: error handling, rollback/recovery, data migration, backward compatibility, observability/logging, security, performance, concurrency.
- Are edge cases and failure modes addressed, or only the happy path?
- Is there a testing/verification strategy? How will "done" be confirmed?
- Open questions and unknowns: are they called out, or silently assumed away?
- **Seam ownership** — when the plan decomposes work across multiple components/steps/tickets, is every cross-piece data dependency (fetch, shared state, persistence/sync, validation contract) assigned to an owning step? An unowned capability that spans pieces is **Critical** when it means a core flow won't work.
- **End-to-end acceptance** — beyond per-step verification, does the plan define one full happy-path acceptance exercising the whole flow (fill real data → act → assert the real outcome)? Flag a plan that only validates pieces in isolation.
- **Acceptance asserts the real state** — flag acceptance criteria that would pass on a degraded/empty/fallback state ("renders results OR the empty state" is not acceptance).
- **Design fidelity** — when the plan implements a referenced design, are the exact tokens (font, size, weight, line-height, hex colors, spacing, dimensions) captured as criteria? "Match the mockup" is unverifiable and leads to reactive pixel churn.

### Correctness & Feasibility

- Do the technical assumptions hold? Verify claims about current behavior against the actual code/config — flag anything the plan asserts that the codebase contradicts.
- Does the proposed approach actually solve the stated problem, or only its symptoms?
- Are there steps that won't work as written (wrong API, non-existent file/method, invalid sequencing, race conditions)?
- Are dependencies between steps real and respected? Does any step depend on a later one?

### Architecture & Design Fit

- Is the approach consistent with the existing system's patterns, layering, and conventions? Does it fight the grain of the codebase?
- **DRY** — does it reinvent a capability the system already has? Could it reuse an existing component/utility/service?
- **KISS / YAGNI** — is it over-engineered? Speculative abstractions, premature generalization, or scope beyond the stated goal?
- Does it introduce tight coupling or a boundary violation that will make future change harder?
- Is there a simpler approach that meets the same goal with less risk or surface area?

### Risk & Edge Cases

- What breaks if a step fails halfway? Is the plan safely resumable / reversible?
- Backward compatibility: existing data, callers, APIs, persisted state, feature flags.
- Security: new attack surface, auth/authz changes, secret handling, input validation.
- Performance & scale: N+1, large-data behavior, hot paths, added latency.
- Concurrency: shared state, ordering, idempotency of steps that may run more than once.

### Scope & Sequencing

- Is the plan right-sized — neither over-scoped (gold-plating, unrelated changes) nor under-scoped (missing necessary work)?
- Can it ship incrementally / behind a flag, or is it an all-or-nothing big-bang?
- Are steps ordered so the system is in a working state between them where possible?
- Is the effort proportionate to the value? Flag steps that add cost without clear payoff.

### Clarity & Actionability

- Is each step concrete and unambiguous enough that an implementer (human or agent) could execute it without guessing intent?
- Are acceptance criteria / definition of done explicit?
- Are file paths, component names, and interfaces named specifically rather than gestured at?
- Is the document internally consistent (no contradictory steps, no stale sections left from earlier drafts)?

### Report Notes (plan-specific)

Use the shared report format in `common.md`, with these adaptations:

- **Principle** line: use `Completeness | Correctness | Feasibility | Architecture | DRY | KISS | Risk | Scope | Clarity` instead of the code principles.
- **File** line: cite the plan's section/heading (and line number if useful), e.g. `plan.md › "Phase 3: Migration"`.
- **Severity:**
  - **Critical** — a flaw that will make the plan fail or produce a broken/incorrect result: wrong approach, missing essential step, false assumption about current behavior, or a change that breaks existing functionality.
  - **Warning** — a gap or ambiguity that risks rework, confusion, or an avoidable bug (missing edge case, unhandled failure, under-specified step).
  - **Info** — a clarification or improvement that would strengthen the plan but isn't load-bearing.
- Include an **Open Questions** subsection when the plan leaves decisions unmade that the author must resolve before implementation.
