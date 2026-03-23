# /audit — Full Project Health Check

Deploy the full specialist team to assess the current state of the codebase across security, performance, accessibility, and code quality. Produces a prioritised remediation roadmap.

## When to use

- Before a major release or launch.
- When onboarding a new team member and you want to document known debt.
- When the codebase has grown without structured review.
- When a specific incident (security, outage, accessibility complaint) prompts a wider look.

## Audit dimensions

Each dimension runs independently and produces its own findings. The final output is a merged, prioritised roadmap.

### 1. Security audit (security-engineer)

- OWASP Top 10 coverage: work through all ten categories explicitly.
- Authentication and session management review.
- Input validation and injection surface.
- Secrets in source: scan git history as well as current state.
- Dependency vulnerabilities: flag any known CVEs in the dependency tree.
- HTTP security headers: check existing middleware or server config.

### 2. Performance audit (performance-engineer)

- Backend: identify N+1 query patterns, missing indexes, long-held transactions.
- Frontend: bundle size analysis, render bottlenecks, unvirtualised lists.
- Caching: identify expensive operations with no caching strategy.
- Database: review schema for common performance anti-patterns.
- Infrastructure: identify any obvious scaling constraints (not infra-specific; flag for ops team).

### 3. Accessibility audit (accessibility-pm)

- Keyboard navigation: can all user journeys be completed without a mouse?
- Screen reader compatibility: semantic HTML, ARIA labels, live regions.
- Visual: colour contrast on key UI elements, focus indicators.
- Forms: label associations, error message patterns.
- Motion: `prefers-reduced-motion` support.
- Output mapped to WCAG 2.1 AA criteria.

### 4. Code quality audit (reviewer)

- Structural consistency: do new code areas follow established patterns?
- Test coverage: what public surface area has no tests?
- Duplication: identify the top three most impactful deduplication opportunities.
- Technical debt: flag areas where complexity has outgrown the current structure.
- Documentation gaps: key systems or flows with no documentation.

## Output format

### Per-dimension findings
```
[CRITICAL/HIGH/MEDIUM/LOW] <dimension> | <finding>
File: <location if applicable>
Impact: <who or what is affected>
Fix: <concrete remediation>
Effort: <XS | S | M | L | XL>
```

### Merged remediation roadmap

Group findings into three tracks:

**Fix now (CRITICAL + HIGH)**
Address before the next release. Ordered by risk.

**Fix next sprint (MEDIUM)**
Schedule in the next iteration. Ordered by effort-to-impact ratio.

**Backlog (LOW + NIT)**
Logged and tracked. No immediate urgency.

### Executive summary (3–5 bullets)
The headline state of the codebase: what is in good shape, what is the most urgent risk, and what is the single highest-leverage improvement.

## Usage

```
/audit                        # full audit of the entire project
/audit --scope security       # single dimension
/audit --scope security,perf  # two dimensions
/audit src/payments/          # scoped to a directory
```

## Notes

- A full audit on a large codebase is a significant session. Consider scoping to a specific module or dimension for faster results.
- Findings are a starting point, not a verdict. Each CRITICAL finding should be triaged by the relevant specialist before remediation begins.
- Schedule re-audits after major changes; debt that was LOW can become HIGH as the system grows.
