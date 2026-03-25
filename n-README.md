# Branching Strategy

> A **Git Flow + Permanent UAT Branch** model with two deployment environments: **UAT** and **Production**.

---

## 1. Branch Structure

| Branch | Purpose | Deploys to | Protected |
|---|---|---|---|
| `main` | Production-ready code. Always stable. | **Production** | Yes |
| `uat` | Permanent UAT environment branch. Reflects exactly what stakeholders are testing. | **UAT** | Yes |
| `develop` | Integration branch. All features land here first. | — | Yes |
| `feature/*` | Individual features or non-urgent fixes. | Local only | No |
| `hotfix/*` | Emergency production fixes only. | UAT (quick verify) | No |

### Branch naming

| Type | Pattern | Example |
|---|---|---|
| Feature | `feature/<short-description>` | `feature/invoice-export` |
| Feature with ticket | `feature/<TICKET-ID>-<description>` | `feature/PROJ-42-user-search` |
| Hotfix | `hotfix/<semver>-<description>` | `hotfix/1.2.1-payment-crash` |

> Lowercase and hyphens only. No spaces, underscores, or special characters.

---

## 2. Environment Map

```
feature/*  →  local only
               │
               │  PR approved → merge
               ▼
           develop   (integration — all features accumulate here)
               │
               │  PR approved → merge → auto deploy
               ▼
            uat       (permanent — tagged uat-X.Y.Z on every merge)
               │
               │  PR approved after stakeholder sign-off → merge → auto deploy
               ▼
            main      (production — tagged vX.Y.Z on every merge)
```

---

## 3. Flow Diagrams

### 3.1 Feature Flow

Branch from `develop`, work, open a PR back to `develop`.

```
develop ──┬──────────────────────────────┬──▶ develop
          │                              │
          └─▶ feature/your-work ─────────┘
               (commit, commit, commit)
               PR reviewed + approved
```

### 3.2 UAT Flow

When `develop` is ready for stakeholder testing, open a PR from `develop` to `uat`. On merge, CI/CD deploys to UAT automatically and a tag is created.

```
develop ──────────────────────────────▶ uat (tag: uat-1.2.0)
         PR: "UAT candidate 1.2.0"         │
                                           │  auto deploy
                                           ▼
                                       UAT server
```

If stakeholders find a bug, **fix it on `develop` first** (via a new `feature/*` PR), then open a new PR from `develop` to `uat`. Never commit directly to `uat`.

```
develop ──▶ feature/uat-fix ──▶ develop ──▶ uat (tag: uat-1.2.1)
```

### 3.3 Production Flow

After stakeholder sign-off on UAT, open a PR from `uat` to `main`. On merge, CI/CD deploys to production and a version tag is created.

```
uat (uat-1.2.0) ──────────────────────▶ main (tag: v1.2.0)
                 PR: "Release 1.2.0"         │
                 sign-off confirmed           │  auto deploy
                                             ▼
                                        Production
```

### 3.4 Hotfix Flow

Production emergency. Branch from `uat`, fix it, merge to `uat` for quick test, then merge to both `main` and `develop`.

```
uat ──▶ hotfix/1.2.1-payment-crash ──┬──▶ uat   (quick test)
                                      │
                                      ├──▶ main   (tag: v1.2.1) → deploy
                                      │
                                      └──▶ develop (keep in sync)
```

> Critical: hotfix must merge into **both** `main` and `develop`. Skipping `develop` means the fix is lost in the next release.

---

## 4. Step-by-Step Workflows

### 4.1 Feature Development

```bash
# Sync develop first
git checkout develop
git pull origin develop

# Branch
git checkout -b feature/invoice-export

# Work and commit
git commit -m "feat: add invoice PDF generation"
git commit -m "feat: add download button on orders page"
git commit -m "test: add unit tests for PDF service"

# Push and open PR → target: develop
git push origin feature/invoice-export

# Open PR request in GitHub/GitLab to merge into develop
# After code review and approval — merge PR into develop

# After merge, sync develop
git checkout develop
git pull origin develop

# Clean up feature branch
git branch -d feature/invoice-export
git push origin --delete feature/invoice-export
```

---

### 4.2 UAT Deploy

```bash
# Ensure develop is stable and has everything needed for this UAT cycle
git checkout develop
git pull origin develop

# Open a PR → target: uat
# Title: "UAT candidate 1.2.0"
# After PR approval — merge into uat

# Tag and push to uat
git tag -a uat-1.2.0 -m "UAT build 1.2.0"
git push origin uat --tags
# → CI/CD deploys to UAT automatically

# If a bug is found during UAT testing:
# 1. Fix it on develop via a normal feature/* PR (section 4.1)
# 2. Then open a new PR and merge from develop → uat
# 3. Tag and push to uat
git tag -a uat-1.2.1 -m "UAT build 1.2.1"
git push origin uat --tags
```

> Rule: `uat` is a receiving branch only. Nobody commits directly to it.

---

### 4.3 Production Release

```bash
# UAT sign-off received
# Open a PR → target: main
# Title: "Release 1.2.0"

# After PR approval — merge into main

# Tag and push to main
git tag -a v1.2.0 -m "Release version 1.2.0"
git push origin main --tags
# → CI/CD deploys to Production
```

---

### 4.4 Hotfix

```bash
# Branch from main (NOT develop)
git checkout main
git pull origin main
git checkout -b hotfix/1.2.1-payment-crash

# Fix
git commit -m "fix: handle null user session in payment gateway"
git push origin hotfix/1.2.1-payment-crash

# Open a PR and merge from hotfix/1.2.1-payment-crash → uat
# Tag and push to uat
git tag -a uat-1.2.1 -m "UAT build 1.2.1"
git push origin uat --tags

# After smoke test passes:

# Open a PR and merge to main
# Tag and push to main
git tag -a v1.2.1 -m "Hotfix 1.2.1"
git push origin main --tags

# ***CRITICAL***: also merge to develop

# Clean up
git branch -d hotfix/1.2.1-payment-crash
git push origin --delete hotfix/1.2.1-payment-crash
```

---

## 5. Rollback Procedures

### Option A — Revert last release (recommended)

Safe, non-destructive. Creates a new revert commit — no history lost.

```bash
git log --oneline main | head -10
git checkout main
git revert -m 1 <merge-commit-sha>
git push origin main
# → CI/CD redeploys main → Production restored
```

### Option B — Redeploy a previous tag (fastest)

Use when you need to jump back to a known-good version immediately.

```bash
git tag --sort=-version:refname | head -10
git checkout v1.1.0
git checkout -b hotfix/emergency-rollback
git push origin hotfix/emergency-rollback
# Follow the hotfix flow: UAT verify → main → uat → develop
```

### Option C — Rollback UAT only

Production is fine, but UAT needs to revert.

```bash
# Retrigger the previous uat tag pipeline, or:
git checkout uat
git revert -m 1 <bad-merge-sha>
git push origin uat
```

### Decision table

| Scenario | Use |
|---|---|
| Bad release on Production | Option A — `git revert -m 1` |
| Need instant prod rollback | Option B — redeploy previous tag |
| UAT broken only | Option C — revert on uat |

---

## 6. Tag Convention

All tags must be following semantic versioning pattern.

`Major.minor.patch`



| Tag | Branch |
|---|---|
| `uat-1.2.0` | `uat` |
| `uat-1.2.1` | `uat` |
| `v1.2.0` | `main` |
| `v1.2.1` | `main` |

Tags are the audit trail. Given any tag, you can always check out the exact code that was running at that point.

```bash
git checkout uat-1.2.0    # what stakeholders approved
git checkout v1.2.0       # what went to production
```

---

## 7. Branch Protection Rules

| Branch | Rules |
|---|---|
| `main` | Require PR · Require 1 approval · Block direct push · Block force push |
| `uat` | Require PR · Block direct push · Block force push |
| `develop` | Require PR · Require 1 approval · Block direct push |

---

## 8. Golden Rules

1. **Never commit directly** to `main`, `uat`, or `develop` — always via PR.
2. **`uat` only receives from `develop` or `hotfix/*`** — never from feature branches directly.
3. **Hotfix merges into three branches**: `main`, `uat`, and `develop`.
4. **Fix UAT bugs on `develop` first**, then re-PR to `uat`.
5. **Tag every merge** to `uat` and `main` — no exceptions.
6. **Delete feature and hotfix branches** after merging for cleaner repository.
