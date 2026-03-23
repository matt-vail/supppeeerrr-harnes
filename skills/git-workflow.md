---
name: git-workflow
description: >
  Git branching strategy, PR workflow, and version control conventions for
  feature delivery. Use when creating branches, opening PRs, handling merge
  conflicts, or integrating code changes into the delivery pipeline.
---

# Skill: git-workflow

Version control conventions and PR workflow for feature delivery.

---

## Branch naming

| Type | Pattern | Example |
|------|---------|---------|
| Feature | `feature/TICKET-ID-short-description` | `feature/PROJ-42-add-jwt-auth` |
| Bug fix | `fix/TICKET-ID-short-description` | `fix/PROJ-87-null-pointer-on-logout` |
| Hotfix | `hotfix/TICKET-ID-short-description` | `hotfix/PROJ-91-critical-auth-bypass` |

**Never commit directly to `main` or `master`.** All changes enter through a branch and a pull request.

---

## Branch lifecycle

- Create a branch before writing a single line of code — never work directly on main.
- One branch per ticket or feature. Do not bundle unrelated changes into one branch.
- Keep branches short-lived. Merge or close within the sprint. A branch open for more than two weeks is a warning sign.

---

## Commit conventions

- **Imperative mood:** "Add user auth" not "Added user auth" or "Adding user auth."
- **Reference the ticket ID:** `feat(auth): add JWT validation [TICKET-123]`
- **Atomic commits:** one logical change per commit. A commit should be independently revertable without breaking other commits.

### Commit type prefixes

| Prefix | Use for |
|--------|---------|
| `feat` | New feature |
| `fix` | Bug fix |
| `refactor` | Code restructuring with no behaviour change |
| `test` | Adding or updating tests |
| `docs` | Documentation only |
| `chore` | Build tooling, dependency updates |
| `perf` | Performance improvement |

---

## Pull request protocol

- **Open a PR as soon as the first commit is pushed.** Use a draft PR if work is still in progress — this makes the branch visible and invites early feedback.
- **PR title** matches the branch purpose and is written in imperative mood: "Add JWT validation for API endpoints."
- **PR description must include:**
  - What changed
  - Why it changed (link to ticket or context)
  - How to test it (steps a reviewer can follow)
  - Screenshots if the change affects the UI

- **Required before merge:**
  - All CRITICAL and HIGH findings from `reviewer` are resolved.
  - CI passes — all lint, test, security, and build stages are green.
  - At least one approving review from a team member.

---

## Merge strategy

| Context | Strategy | Reason |
|---------|----------|--------|
| Feature branch → main | **Squash merge** | Keeps main history clean — one logical commit per feature |
| Release branch → main | **Merge commit** | Preserves full release history for audit trail |

**Never force-push to `main` or `master`.** Force-push to personal feature branches is acceptable before the PR is reviewed; after review begins, coordinate with reviewers before rewriting history.

---

## Conflict resolution

- Rebase your feature branch onto main before opening a PR when possible — this produces a clean, linear history and surfaces conflicts before the reviewer sees the branch.
- Resolve merge conflicts locally, not in the PR UI, unless the conflict is trivial (a comment change, whitespace).
- If the conflict involves logic owned by another domain: surface to the specialist who owns that domain before resolving. Do not guess at intent.

---

## Hard situations

**Long-running branch diverged significantly from main:**
Rebase in stages. Identify natural breakpoints (sub-features, layers) and rebase incrementally. Attempting to resolve a divergence of hundreds of commits in one pass produces conflicts that are impossible to reason about.

**Hotfix needed on a release branch:**
Branch from the release tag, not from main. Main may contain unreleased work that must not enter the hotfix. After the hotfix is merged and tagged, cherry-pick or merge forward to main.

**PR is blocked on a dependency from another team:**
Open a draft PR to make the work visible. Document the dependency in the PR description: "Blocked on [team/PR link] — waiting for [specific thing]." Flag the blocker to `orchestrator` so it can be tracked. Do not leave the dependency undocumented.

**Reviewer has requested changes but the branch has since fallen behind main:**
Resolve reviewer comments first, then rebase onto the updated main. Rebasing before addressing comments can silently resolve comment threads, making it appear the feedback was addressed when it was not.
