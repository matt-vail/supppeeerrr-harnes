# /dependencies — Dependency Audit

Route this task to `dependency-engineer`. Audit the project's dependency tree for known CVEs, licence conflicts, outdated packages, and abandoned dependencies. Produce an actionable findings report with exact remediation commands.

## When to use

- Before a release to verify no new CVEs have entered the dependency tree.
- When adopting a new dependency or upgrading a major version.
- When a CVE advisory is published affecting a package in the ecosystem.
- When a new project is being onboarded and its dependency health is unknown.
- Periodically as a scheduled hygiene check — dependency risk accumulates silently.

## Usage

```
/dependencies                         # full dependency audit — all dimensions
/dependencies --scope security        # CVEs only
/dependencies --scope licences        # licence compliance only
/dependencies --scope outdated        # version drift only
/dependencies --fix                   # produce upgrade plan with ordered commands
```

### Scope flags

| Flag | What runs |
|------|----------|
| *(none)* | Full audit: CVEs, licences, outdated packages, abandoned packages, dependency count health |
| `--scope security` | CVE scan only — runs `npm audit` / `pip-audit` / `govulncheck` / `cargo audit` / `dotnet list package --vulnerable` as appropriate |
| `--scope licences` | Licence compliance only — lists all dependency licences and flags conflicts |
| `--scope outdated` | Version drift only — identifies packages behind current stable with migration notes |
| `--fix` | Appends a full upgrade plan to any audit output — ordered upgrade commands and breaking-change notes |

`--fix` can be combined with any `--scope` flag: `/dependencies --scope security --fix` produces CVE findings plus the commands to resolve them.

---

## Output sections

### 1. CVE findings

One finding block per vulnerable package, formatted using `skills/findings-template.md`.

```
[{LEVEL}] CVE — {CVE-ID}
Location:  {package-name}@{installed-version}
Asset:     {what is at risk — data, process integrity, supply chain}
Finding:   {description of the vulnerability — attack vector, CVSS score, affected range}
Fix:       {exact shell command to upgrade or override the dependency}
Verify:    {re-run audit command; expected output showing vulnerability resolved}
```

CRITICAL findings (CVSS ≥ 9.0) are escalated to `security-engineer` before any other action. The audit report will note this escalation and return `PARTIAL` status until the security assessment is complete.

### 2. Licence report

One finding block per flagged licence condition.

```
[{LEVEL}] Licence — {SPDX-ID}
Location:  {package-name}@{version} (direct | transitive via {parent})
Asset:     {legal exposure — copyleft contamination, commercial SaaS compliance, etc.}
Finding:   {why this licence is problematic — specific conflict or restriction}
Fix:       {replace with {alternative-package} | obtain commercial licence | open-source project}
Verify:    {re-run licence check; confirm flagged package no longer present in tree}
```

After the finding blocks, include a **Licence summary table** listing all dependency licences grouped by SPDX identifier — this gives a full picture of the licence surface, not just the flagged items.

### 3. Outdated packages

One finding block per outdated package at MEDIUM or above, plus a grouped LOW summary table for minor drift.

```
[{LEVEL}] Version drift — {package-name}
Location:  {package-name}@{installed-version} → latest: {latest-version}
Asset:     {security, stability, or compatibility}
Finding:   {major versions behind | security patch in gap — list affected versions}
Fix:       {exact upgrade command; link to migration guide if major version bump}
Verify:    {re-run outdated check; confirm package at or near latest}
```

### 4. Abandoned packages

One finding block per abandoned package.

```
[MEDIUM] Abandoned — {package-name}
Location:  {package-name}@{installed-version}
Asset:     {future security and maintenance — no patched versions will be published}
Finding:   {last published: {date} — {N} months/years ago; no maintainer activity detected}
Fix:       {replace with {alternative} | vendor the dependency with documented ownership}
Verify:    {package replaced or vendored; no remaining import of abandoned package}
```

### 5. Upgrade plan (--fix only)

An ordered list of upgrade commands grouped by risk level. Apply in this order: CRITICAL CVEs first, then HIGH, then MEDIUM, then LOW. Breaking changes are called out explicitly before their command.

```
## Upgrade plan

### Critical CVE remediations (apply immediately)
1. {package-name}: {npm install {pkg}@{version} | pip install {pkg}=={version} | ...}
   CVE: {CVE-ID} — resolves after upgrade to {version}

### High — security patches and major drift
2. {package-name}: {upgrade command}
   Breaking changes: {list key breaking changes if major version}
   Migration guide: {URL if available}

### Medium — minor drift with security patches
3. ...

### Low — maintenance upgrades (non-urgent)
4. ...

### Post-upgrade verification
Run: {full audit command for the ecosystem}
Expected: no vulnerabilities at HIGH or above
```

---

## Summary line

After all sections, include the standard findings summary:

```
Summary: {N} CRITICAL, {N} HIGH, {N} MEDIUM, {N} LOW — dependency audit ({ecosystems audited})
```

Then offer to apply fixes for any CRITICAL or HIGH findings directly.

---

## Integration with /audit

`/dependencies` runs automatically as a fifth dimension in `/audit`, alongside the security, performance, accessibility, and code quality audits. When running as part of `/audit`, its findings are merged into the unified remediation roadmap under a **Dependency health** track.

When `/audit` and `/dependencies` both surface CVE findings, they are de-duplicated — `security-engineer`'s supply chain findings and `dependency-engineer`'s CVE scan are reconciled before the roadmap is produced.

---

## Integration with /ship

**CRITICAL CVE findings from `/dependencies` block `/ship` completion.** A `/ship` run that encounters unresolved CRITICAL CVEs in the dependency tree will halt at the dependency gate and return a blocked status. The gate will not clear until:
1. The CRITICAL CVE has been remediated (package upgraded or overridden), or
2. `security-engineer` has assessed the finding and a human has explicitly accepted the risk with a documented rationale.

HIGH CVE findings surface as warnings in `/ship` output but do not block. They are included in the pre-ship checklist for human review.

---

## Notes

- Audit results depend on the audit database being current. For `npm audit`, the database is fetched at run time. For `pip-audit`, ensure `pip-audit` is up to date. For `govulncheck`, ensure the Go vulnerability database is reachable.
- Transitive vulnerability remediation is harder than direct — if a direct parent has no patched version, `dependency-engineer` will surface resolution override options and flag any cases with no viable path.
- Lock file hygiene is checked as part of every `/dependencies` run. A missing lock file is itself a finding.
- For projects with compliance requirements, ask `dependency-engineer` to produce an SBOM (`cyclonedx-npm`, `cyclonedx-python`, `syft`) as a separate artefact — flag to `tech-writer` to document as an ADR.
