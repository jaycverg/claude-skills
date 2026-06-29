# SvelteKit / TypeScript Review Criteria

Applies to `.svelte`, `.ts`, `.js`, `.css`, `.json` (config files).

**Context first:** for each changed file, read the full diff and surrounding code, and identify whether files are pages, layouts, server routes (`+server.ts`), load functions (`+page.server.ts`, `+page.ts`), components, or stores.

Evaluate against the categories below. Report only actionable findings — skip categories where the change is clean.

### SOLID Principles

| Principle | What to Check |
|-----------|--------------|
| **S** — Single Responsibility | Does each component/function do one thing? Are components mixing data loading, business logic, and presentation? |
| **O** — Open/Closed | Are modifications extending via composition (slots, snippets, component props), or editing existing logic that should be closed? |
| **L** — Liskov Substitution | Do component variants honor the base component's prop contract? |
| **I** — Interface Segregation | Are prop types lean, or forcing consumers to pass unused props? Are shared types bloated? |
| **D** — Dependency Inversion | Do modules depend on abstractions (interfaces, type contracts) or concrete implementations? |

### DRY (Don't Repeat Yourself)

- Duplicated logic across the diff or between the diff and existing code
- Copy-pasted markup blocks that should be a shared component
- Repeated fetch/API logic that should be in a load function or shared utility
- Duplicated type definitions that should be shared
- Repeated string literals, magic numbers, or config values that belong in constants
- Note: three similar lines can be better than a premature abstraction — flag only clear duplication

### KISS (Keep It Simple, Stupid)

- Over-engineered solutions (unnecessary stores for simple local state, excessive abstraction layers)
- Complex reactive statements (`$:` / `$derived`) that could be simplified
- Premature generalization or speculative future-proofing
- Using stores when simple prop passing or context would suffice

### Clean Code

- **Naming**: Are components, variables, and functions named clearly? Stores suffixed with convention?
- **Components**: Small, focused, single responsibility? Reasonable prop count?
- **Comments**: Explaining *why*, not *what*? Missing where logic is non-obvious?
- **Error handling**: Appropriate use of `+error.svelte` pages, try/catch in load functions and form actions, loading states?

### Architecture & Design (SvelteKit-Specific)

- Are load functions used correctly (`+page.server.ts` for server-only data, `+page.ts` for universal loads)?
- Is form handling using SvelteKit form actions instead of manual API calls where appropriate?
- Is state managed at the right level (component state vs stores vs URL state vs server state)?
- Are `+layout.ts`/`+layout.server.ts` used correctly for shared data loading?
- Does the change respect SvelteKit's file-based routing conventions (`+page.svelte`, `+layout.svelte`, `+error.svelte`, `+server.ts`)?
- Are side effects isolated and predictable?
- Does the change introduce coupling that will make future changes harder?
- Are API routes (`+server.ts`) handling validation and errors properly?
- Is `$env/static/private` vs `$env/static/public` used correctly (no server secrets in client code)?

### Code Quality (TypeScript/Svelte-Specific)

- **Type safety**: Are types used correctly? Flag any use of `any` — prefer `unknown`, specific types, or generics. Avoid unsafe type assertions (`as`). Ensure load function return types and prop types are explicit where non-obvious.
- **Reactivity**: Are reactive declarations (`$:` / `$derived`, `$effect`) used correctly? Unnecessary reactivity or missing reactive updates? Are stores subscribed/unsubscribed properly (auto-subscription `$store` preferred)?
- **State management**: Derived state kept as reactive computation, not duplicated? Writable stores used only when mutation is needed?
- **Edge cases**: Boundary conditions handled (empty arrays, null/undefined, loading states)?
- **Security**: XSS risks (`{@html}`), exposed secrets in client code (`$env/static/public` vs `private`), missing input validation on form actions/API routes?
- **Performance**: Unnecessary reactivity triggering, large client bundles, missing code splitting, unoptimized images, N+1 data fetching in load functions?
- **Testability**: Is the changed code testable? Are side effects isolated? Are components pure where possible?

### Svelte 3 / Sapper & SSR landmines

These apply when the diff is on a legacy **Svelte 3 / Sapper** stack as well as SvelteKit. Flag:

- **`:global()` wrapping a functional pseudo-class** — any `:has()` / `:not()` (or other functional pseudo-class) inside `:global(...)`: Svelte 3/Sapper **silently drops the entire component stylesheet** at compile — the build succeeds and ships zero CSS for that component. Use compound selectors instead (`label.required`). `:not()` is fine in ordinary scoped rules; the trap is specifically inside `:global()`.
- **Service worker missing `skipWaiting()`** — a Sapper/SW lacking `skipWaiting()` (paired with `clients.claim()`) leaves new deploys "waiting" and serves stale bundles, so fixed bugs reappear after a deploy. Flag a service worker missing them.
- **Shallow merge over nested state** — a shallow `{...}` / `Object.assign` that replaces a nested object wholesale drops the nested object's defaults (e.g. a postData↔cache merge clobbering `client.notifyBySms` / `notifyByEmail` / `releaseContactInfo`). Flag a spread/assign that overwrites a nested object the consumer still expects to carry defaults.
- **SSR↔client hydration parity** — an `{#if}` whose branch differs between SSR and the client's initial render (e.g. data status LOADED on the server vs PENDING on the client) duplicates DOM (the double "Add Category" CTA). Gate on a `mounted` flag so both renders agree.
- **Build-success ≠ styles shipped** — for Svelte/Sapper, a passing build is not proof the component CSS shipped (see `:global()` trap above); expect the change to be verified against rendered output, not the build exit code.

(`{@html}` XSS is covered under Security above — reinforce that escaping is required for any interpolated user/derived content.)
