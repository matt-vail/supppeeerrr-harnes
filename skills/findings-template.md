---
name: findings-template
description: Standard format for structured findings across security, accessibility, performance, and code review. Activates when reporting issues, vulnerabilities, audit results, or any prioritised list of problems found.
triggers:
  - finding
  - findings
  - vulnerability
  - issue found
  - audit result
  - problems found
  - review results
  - security issue
  - accessibility issue
  - performance issue
  - report
---

# Skill: findings-template

All agents that produce findings — security reviews, accessibility audits, performance profiling, code reviews — use this standard format. Consistent structure means findings can be compared, prioritised, and tracked across agents and sessions.

---

## Priority levels

| Level | Meaning | Action required |
|-------|---------|----------------|
| **CRITICAL** | Exploitable, data loss, system down, or fundamental correctness failure | Block merge / deployment. Fix before anything else. |
| **HIGH** | Significant risk or quality failure — likely to cause a real problem | Fix before merge. Can proceed to staging but not production. |
| **MEDIUM** | Notable debt or risk — will cause a problem if left long enough | Fix in the next iteration. Track in backlog. |
| **LOW** | Minor issue — low probability or low impact | Fix in a follow-up. Non-blocking. |
| **NIT** | Style or preference — not a standards violation | Optional. Label as preference if subjective. |

---

## Finding format

Each finding is one block:

```
[{LEVEL}] {Category or standard reference}
Location:  {file:line or component name — be specific}
Asset:     {what is at risk — data, user, system, correctness}
Finding:   {what the code or system does wrong — factual, not opinion}
Fix:       {concrete remediation — include a code snippet or specific change if possible}
Verify:    {how to confirm the fix is effective — test, command, or observable behaviour}
```

**Category examples by domain:**
- Security: `OWASP A03 Injection`, `CWE-89`, `JWT validation`
- Accessibility: `WCAG 1.3.1 Info and Relationships`, `POUR — Perceivable`
- Performance: `N+1 query`, `Bundle size`, `Web Vitals — LCP`
- Code review: `Correctness`, `Clarity`, `Test coverage`, `Structural`

---

## Findings list structure

Present findings in this order:

1. All CRITICAL findings first
2. Then HIGH, MEDIUM, LOW, NIT
3. Within each level: order by likelihood of user/system impact

After the list, include a one-line summary:
```
Summary: {N} CRITICAL, {N} HIGH, {N} MEDIUM, {N} LOW, {N} NIT — {dominant lens or category}
```

Then offer to apply fixes for any CRITICAL or HIGH findings directly.

---

## Rules

- Every finding has a specific location. "The auth module" is not a location. `src/auth/jwt.py:42` is.
- Every finding has a concrete fix. "Improve validation" is not a fix. "Add `if not isinstance(token, str): raise ValueError` at line 42" is.
- Distinguish facts from opinions. A standards violation is a fact — cite the standard. A preference is an opinion — label it as NIT with "preference:".
- Do not include findings you are not confident in. If uncertain, note it: "Possible [LEVEL] — needs investigation: [what to check]."

---

## Graph entries

CRITICAL and HIGH findings that trace to a specific task, decision, or prior incident should be added to `agent-memory/graph.md` as `finding` nodes, with edges to the tasks that surface or resolve them.

```
| FINDING-{NNN} | finding | {agent-name} | {one-line description} | {date} |
```
