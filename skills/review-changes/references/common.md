# Review — Shared Scope & Report Format

Used by every framework path of the `review-changes` skill.

## Diff Scope

Resolve the review scope from the skill arguments:

| Argument       | Scope                                                       |
|----------------|-------------------------------------------------------------|
| _(none)_       | all uncommitted changes (`git diff` + `git diff --cached`)  |
| File path(s)   | changes in those files (`git diff <path>`)                  |
| Commit hash    | that commit (`git show <hash>`)                             |
| Commit range   | the range (`git diff <from>..<to>`)                         |
| Branch name    | branch diff vs base (`git diff <base>..HEAD`)               |
| `--staged`     | only staged changes (`git diff --cached`)                   |
| `--full`       | all files in the project, not just changes                  |

For branch scope, `<base>` is the repo's default branch — detect it (e.g. `git symbolic-ref --short refs/remotes/origin/HEAD`, falling back to `main`/`master`) rather than assuming `main`.

If no changes exist in the requested scope, inform the user and stop.

## Report Format

Structure findings as:

```
## Review: <scope description>

### Summary
<1-3 sentence overview of change quality>

### Findings

#### [Critical] <title>
**File:** `path/to/file:line`
**Principle:** SOLID/S | DRY | KISS | Clean Code | Architecture | Quality
**Issue:** <description>
**Suggestion:** <concrete fix>

#### [Warning] <title>
...

#### [Info] <title>
...

### What's Done Well
<Highlight good patterns, clean implementations, or improvements>
```

**Severity levels:**
- **Critical** — bugs, security issues, architectural violations that will cause problems
- **Warning** — code smells, principle violations that degrade maintainability
- **Info** — minor suggestions, style preferences, improvement opportunities

**Rules:**
- Always include file paths and line numbers
- Provide concrete code suggestions, not vague advice
- Every finding must have a clear "Suggestion"
- Include "What's Done Well" — reviews should be constructive
- If changes are clean with no findings, say so explicitly
- Order findings by severity (critical first)
