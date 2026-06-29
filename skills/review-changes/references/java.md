# Java Review Criteria

Applies to `.java`, `.xml` (pom.xml, Spring configs), `.properties`, `.yml`/`.yaml`.

**Context first:** for each changed file, read the full diff and surrounding code, and identify the framework/patterns in use (Spring Boot, Jakarta EE, etc.).

Evaluate against the categories below. Report only actionable findings — skip categories where the change is clean.

### SOLID Principles

| Principle | What to Check |
|-----------|--------------|
| **S** — Single Responsibility | Does each class/method do one thing? Are changes mixing unrelated concerns? Are services handling both business logic and persistence? |
| **O** — Open/Closed | Are modifications extending via abstraction (strategy, decorator), or editing existing logic that should be closed? |
| **L** — Liskov Substitution | Do subclass/interface implementations honor the parent contract? Are overridden methods changing expected behavior? |
| **I** — Interface Segregation | Are interfaces lean, or forcing consumers to depend on unused methods? Are fat service interfaces being created? |
| **D** — Dependency Inversion | Are high-level modules depending on abstractions (`@Autowired` on interfaces, not concrete classes)? Is constructor injection used over field injection? |

### DRY (Don't Repeat Yourself)

- Duplicated logic across the diff or between the diff and existing code
- Copy-pasted blocks that should be a shared utility or base class method
- Repeated string literals, magic numbers, or config values that belong in constants or `application.properties`
- Duplicate exception handling blocks that could use a common handler
- Note: three similar lines can be better than a premature abstraction — flag only clear duplication

### KISS (Keep It Simple, Stupid)

- Over-engineered solutions (unnecessary design patterns, excessive generics)
- Unnecessary abstractions, indirection layers, or wrapper classes
- Complex stream chains that would be clearer as a simple loop
- Premature generalization or speculative future-proofing

### Clean Code

- **Naming**: Are classes, methods, and variables named clearly using Java conventions (camelCase for methods/variables, PascalCase for classes)?
- **Methods**: Short, focused, single level of abstraction?
- **Comments**: Explaining *why*, not *what*? Javadoc on public APIs where non-obvious?
- **Error handling**: Appropriate use of checked vs unchecked exceptions? Not swallowing silently, not catching `Exception` or `Throwable` broadly?

### Architecture & Design

- Does the change respect existing layer boundaries (controller → service → repository)?
- Are dependencies flowing in the right direction?
- Is state managed appropriately (stateless services, thread safety)?
- Are side effects isolated and predictable?
- Does the change introduce tight coupling that will make future changes harder?
- Are transactional boundaries correct (`@Transactional` scope)?

### Code Quality (Java-Specific)

- **Null safety**: Proper use of `Optional`, null checks at boundaries, `@Nullable`/`@NonNull` annotations?
- **Generics**: Raw types avoided? Wildcards used correctly (`? extends` vs `? super`)?
- **Concurrency**: Thread safety considered? Mutable shared state guarded? `synchronized`, `volatile`, or concurrent collections used correctly?
- **Resource management**: Are streams, connections, and I/O closed properly (try-with-resources)?
- **Edge cases**: Boundary conditions handled (empty collections, null inputs at API boundaries)?
- **Security**: SQL injection (use parameterized queries), XSS, exposed secrets, insecure deserialization, missing input validation on controller endpoints?
- **Performance**: N+1 queries, unnecessary eager fetching, O(n^2) where O(n) is possible, excessive object creation in hot paths?
- **Testability**: Is the changed code testable? Are dependencies injectable via constructor?
