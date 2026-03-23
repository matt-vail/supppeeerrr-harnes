# /docs — Documentation Generation & Improvement

Activate the `tech-writer` specialist to generate, audit, or improve documentation for the specified target.

## When to use

- After a feature is complete and documentation hasn't been written yet.
- When a README or API doc is outdated relative to the current code.
- When inline comments are missing from a complex or non-obvious function.
- When preparing for a public API release or team expansion.
- When an ADR needs to be written for a recent architectural decision.

## Modes

### `--readme`
Audit or generate the project README. Checks:
- Does it explain what the project does in one paragraph without jargon?
- Are setup instructions accurate and complete for a clean machine?
- Are test and deployment commands present?
- Are all links valid?

Produces a complete, updated README if missing or significantly outdated.

### `--api`
Generate or update API documentation for the specified endpoints or module:
- OpenAPI 3.1 spec if one doesn't exist, or update the existing spec.
- Consumer-facing prose guide: authentication, key endpoints, request/response examples, error codes.
- Rate limits and quota information if applicable.

### `--inline`
Review and improve inline code comments in the specified file(s):
- Delete comments that explain *what* — the code already says that.
- Add comments that explain *why* for non-obvious logic, business rules, and workarounds.
- Add or improve docstrings on all public functions and classes (detect the project's docstring style and match it).

### `--adr`
Produce an Architecture Decision Record for a recent or upcoming technical decision:
- Prompts for: context, options considered, decision made, consequences.
- Saves to the project's ADR directory (`docs/adr/`, `docs/decisions/`, or detected equivalent).
- Numbered sequentially from existing ADRs.

### `--changelog`
Update `CHANGELOG.md` based on recent commits or a specified diff:
- Follows Keep a Changelog format.
- Groups changes under: `Added`, `Changed`, `Deprecated`, `Removed`, `Fixed`, `Security`.
- Links each entry to the relevant issue or PR if available.

### `--onboard`
Generate an onboarding guide for a new engineer:
- Domain knowledge primer.
- Codebase map (where the important things live and why).
- Development workflow (branches, PRs, deployment).
- Known gotchas and non-obvious conventions.
- Contacts (requests developer to fill in team info).

## Usage

```
/docs --readme
/docs --api src/routes/users.ts
/docs --inline src/payments/processor.py
/docs --adr "switching from REST to GraphQL for the mobile API"
/docs --changelog
/docs --onboard
```

Multiple modes can be combined:
```
/docs --api --changelog
```

## Output

All generated docs are:
- Ready to commit as-is, or to use as a first draft requiring only domain-specific fill-in.
- Formatted to match the existing documentation style in the project.
- Accurate to the current code state (the `tech-writer` reads the code before writing).

## Notes

- Running `/docs --inline` on a large codebase can take time. Scope it to the files most recently changed.
- ADRs are most valuable when written close to the decision. Don't wait until the decision is ancient history.
- For API docs generated during feature development, prefer running this as the final stage of `/ship` rather than separately.
