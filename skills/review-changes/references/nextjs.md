# Next.js / TypeScript Review Criteria

Applies to `.ts`, `.tsx`, `.js`, `.jsx`, `.css`, `.json` (config files).

**Context first:** for each changed file, read the full diff and surrounding code, and identify whether files are Server Components, Client Components, API routes, middleware, hooks, or utilities.

Evaluate against the categories below. Report only actionable findings — skip categories where the change is clean.

### SOLID Principles

| Principle | What to Check |
|-----------|--------------|
| **S** — Single Responsibility | Does each component/hook/function do one thing? Are components mixing data fetching, business logic, and presentation? |
| **O** — Open/Closed | Are modifications extending via composition (HOCs, hooks, render props), or editing existing logic that should be closed? |
| **L** — Liskov Substitution | Do component variants honor the base component's prop contract? |
| **I** — Interface Segregation | Are prop types lean, or forcing consumers to pass unused props? Are shared types bloated? |
| **D** — Dependency Inversion | Do modules depend on abstractions (interfaces, type contracts) or concrete implementations? |

### DRY (Don't Repeat Yourself)

- Duplicated logic across the diff or between the diff and existing code
- Copy-pasted JSX blocks that should be a shared component
- Repeated fetch/API logic that should be a custom hook or utility
- Duplicated type definitions that should be shared
- Repeated string literals, magic numbers, or config values that belong in constants
- Note: three similar lines can be better than a premature abstraction — flag only clear duplication

### KISS (Keep It Simple, Stupid)

- Over-engineered solutions (unnecessary context providers, excessive abstraction layers)
- Unnecessary custom hooks wrapping simple one-liners
- Complex conditional rendering that could be simplified
- Premature generalization or speculative future-proofing

### Clean Code

- **Naming**: Are components, hooks, and variables named clearly? Hooks prefixed with `use`? Event handlers with `handle`/`on`?
- **Components**: Small, focused, single responsibility? Reasonable prop count?
- **Comments**: Explaining *why*, not *what*? Missing where logic is non-obvious?
- **Error handling**: Appropriate use of error boundaries, try/catch in server actions, loading/error states?

### Architecture & Design (Next.js-Specific)

- Does the change respect the Server/Client Component boundary? Is `"use client"` added only when necessary?
- Are data fetching patterns correct (server components for data, not client-side `useEffect` for initial loads)?
- Is state managed at the right level (local state vs context vs URL state vs server state)?
- Are side effects isolated and predictable?
- Does the change introduce coupling that will make future changes harder?
- Are API routes/server actions handling validation and errors properly?
- Is the App Router used correctly (layouts, loading, error, not-found conventions)?

### Code Quality (TypeScript/React-Specific)

- **Type safety**: Are types used correctly? Flag any use of `any` — prefer `unknown`, specific types, or generics. Avoid unsafe type assertions (`as`). Ensure function return types and prop types are explicit where non-obvious.
- **React patterns**: Correct hook dependencies? Unnecessary re-renders (missing `useMemo`/`useCallback` where it matters)? Rules of hooks followed?
- **State management**: Derived state kept as computation, not duplicated in state? Controlled vs uncontrolled components used correctly?
- **Edge cases**: Boundary conditions handled (empty arrays, null/undefined, loading states)?
- **Security**: XSS risks (dangerouslySetInnerHTML), exposed secrets in client code, missing input validation on API routes/server actions, CSRF considerations?
- **Performance**: Unnecessary re-renders, large client bundles (heavy imports in client components), missing Suspense boundaries, unoptimized images, N+1 data fetching?
- **Testability**: Is the changed code testable? Are side effects isolated? Are components pure where possible?
