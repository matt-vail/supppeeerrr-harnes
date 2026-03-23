# /accessibility — Accessibility Audit & Remediation

Run the `accessibility-pm` specialist against the specified UI code. Produces WCAG-referenced findings and applies fixes directly.

## When to use

- Before shipping any user-facing UI change.
- When an accessibility bug is reported.
- As part of a regular audit cycle.
- When adding new interactive components (modals, dropdowns, forms, data tables).

## Process

### Phase 1 — Scan

The `accessibility-pm` reviews the specified files or current diff through the four POUR lenses:

**Perceivable**
- Missing alt text on images.
- Colour contrast failures.
- Information conveyed by colour alone.
- Content that disappears or moves without warning.

**Operable**
- Keyboard traps or unreachable interactive elements.
- Missing or suppressed focus indicators.
- Incorrect or absent ARIA keyboard patterns for custom widgets.
- Touch targets below 44×44px.

**Understandable**
- Form inputs without associated labels.
- Error messages that don't explain what went wrong or how to fix it.
- Inconsistent navigation behaviour.
- Missing `lang` attribute on the document.

**Robust**
- Missing or incorrect ARIA roles, states, and properties.
- Dynamic content changes not announced via `aria-live`.
- Broken heading hierarchy.
- Missing landmark regions.

### Phase 2 — Report

Each finding is output as:
```
[WCAG X.X.X | CRITICAL/HIGH/MEDIUM/LOW]
Component/file: <location>
Affected users: <who this impacts — e.g. screen reader users, keyboard-only users>
Problem: <what is wrong>
Fix: <before/after code snippet>
Test with: <how to verify — keyboard, VoiceOver, NVDA, axe DevTools, etc.>
```

Summary section:
- Total findings by severity.
- Most impacted user groups.
- Quick wins (high impact, low effort).
- Estimated WCAG 2.1 AA conformance level for the reviewed scope.

### Phase 3 — Remediation

After the report, offer to apply fixes:
- Apply CRITICAL and HIGH fixes immediately.
- For each fix, show the before/after diff.
- Confirm the fix does not break other functionality.
- Re-check the repaired element against the relevant WCAG criterion.

### Phase 4 — Regression test

For each significant fix, add an automated accessibility test where tooling supports it:
- `jest-axe` for React component tests.
- `axe-playwright` or `axe-cypress` for E2E flows.
- A focused keyboard-navigation test for interactive components.

## Usage

```
/accessibility                       # audit current diff
/accessibility <file or glob>        # audit specific files
/accessibility src/components/Modal  # audit a specific component
```

### Examples

```
/accessibility src/components/DropdownMenu.tsx
/accessibility src/pages/checkout/
/accessibility
```

## Notes

- Automated tools catch ~30–40% of accessibility issues. Manual keyboard and screen reader testing catches the rest. This command handles the automated and code-review layer — schedule manual testing for critical flows.
- For a project-wide accessibility assessment, use `/audit --scope accessibility` instead.
- WCAG 2.1 AA is the minimum target. Flag AAA criteria worth pursuing as LOW findings.
