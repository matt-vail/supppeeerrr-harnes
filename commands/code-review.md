# /code-review — Code Review & Refactoring

Perform a structured review of the specified file(s) or diff and produce a prioritised list of findings.

## Review mode (configurable)

- Default: DIFF mode — reviews only changed lines plus 20 lines of surrounding context
- Override: pass --full-files to review entire files regardless of diff

Diff mode behaviour:
1. Run `git diff HEAD` (or `git diff --staged` if changes are staged) to get changed lines.
2. If no git diff is available (untracked files, no git repo), fall back to full-file mode automatically.
3. Pass the diff output + 20 lines surrounding context per hunk to the reviewer.
4. Full file read only if: the diff references a type, function, or class defined in another unchanged file that is critical to understanding the change.
5. State at the start of review output: "Review mode: DIFF (N lines changed across M files)" or "Review mode: FULL FILES (--full-files flag set)"

## Review Dimensions

Analyse the code across these four lenses, in order:

1. **Correctness & Edge Cases**
   - Logic errors, off-by-ones, null/undefined dereferences.
   - Missing error handling or swallowed exceptions.
   - Edge inputs (empty collections, zero, negative numbers, very large values, concurrent access).

2. **Clarity & Readability**
   - Naming that obscures intent.
   - Functions or methods that do more than one thing.
   - Comments that describe *what* instead of *why*.
   - Cognitive complexity that could be simplified.

3. **Structural Concerns**
   - Violations of established project patterns detected in the codebase.
   - Unnecessary coupling or missing abstractions.
   - Duplication that should be extracted.
   - Security smells (hardcoded secrets, unsanitised input, overly permissive access).

4. **Test Coverage Gaps**
   - Happy-path tests that exist but miss edge cases.
   - Untested public surface area.
   - Tests that assert too little or too much.

## Output Format

Return findings as a prioritised list:

```
[CRITICAL] <finding> — <suggested fix>
[HIGH]     <finding> — <suggested fix>
[MEDIUM]   <finding> — <suggested fix>
[LOW]      <finding> — <suggested fix>
[NIT]      <finding> — <suggested fix>
```

After the list, offer to apply any of the fixes directly.

## Usage

```
/code-review [file or glob]
/code-review              # reviews the current diff (staged + unstaged changes)
```

### Examples

```
/code-review src/auth/login.ts
/code-review **/*.test.js
/code-review
```

## Notes

- Delegate to the `reviewer` agent for a full interactive review session.
- Findings are ordered by impact; address CRITICALs before committing.
