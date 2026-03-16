# Git Branching Strategy & Standards

> **Authoritative team reference.** All developers must follow this guide.  
> Sources: [Microsoft Azure DevOps Branching Guidance](https://learn.microsoft.com/en-us/azure/devops/repos/git/git-branching-guidance?view=azure-devops) · [AWS Prescriptive Guidance – Choosing a Git Branch Approach](https://docs.aws.amazon.com/prescriptive-guidance/latest/choosing-git-branch-approach/git-branching-strategies.html)

---

## Table of Contents

1. [Overview & Philosophy](#1-overview--philosophy)
2. [Branch Model](#2-branch-model)
3. [Branch Naming Conventions](#3-branch-naming-conventions)
4. [Workflow: Day-to-Day Development](#4-workflow-day-to-day-development)
5. [Feature Branches](#5-feature-branches)
6. [Release Branches](#6-release-branches)
7. [Hotfix Branches](#7-hotfix-branches)
8. [Tags](#8-tags)
9. [Pull Requests (PRs)](#9-pull-requests-prs)
10. [Squash Commits & Clean History](#10-squash-commits--clean-history)
11. [Branch Protection Rules](#11-branch-protection-rules)
12. [Deleting Stale Branches](#12-deleting-stale-branches)
13. [Commit Message Standards](#13-commit-message-standards)
14. [Quick Reference Cheatsheet](#14-quick-reference-cheatsheet)
15. [Enforcement Setup (GitHub)](#15-enforcement-setup-github)

---

## 1. Overview & Philosophy

We adopt a **Gitflow-inspired strategy** (as advocated by both Microsoft and AWS) blended with GitHub Flow simplicity for day-to-day work. The guiding principles are:

| Principle | Rule |
|-----------|------|
| **Stable branches are sacred** | `main` and `develop` can never be pushed to directly |
| **All work lives in branches** | Every feature, bugfix, hotfix, and release gets its own branch |
| **Code review is mandatory** | All changes reach stable branches via Pull Request only |
| **History must be clean** | Squash-merge all PRs into `main`/`develop` |
| **Releases are explicit** | Releases are created from `develop`, tagged, then merged to `main` |

This strategy was chosen because:
- **Microsoft** recommends feature branches + PR-only merges into main, with release branches for multi-version support
- **AWS** recommends Gitflow for teams managing multiple concurrent releases (our real estate platform has auctions, mortgage, and listings — each with separate release cadences)
- It prevents accidental broken code reaching production
- It creates a clear audit trail for every change

---

## 2. Branch Model

```
main  ──────────────────────────────────────────────────────────►  (production, always deployable)
         ▲  merge via PR (squash)       ▲  merge via PR (squash)
         │                              │
release/1.0 ──────────────────────────►  (stabilisation, bug fixes only)
                     ▲
develop ─────────────────────────────────────────────────────────►  (integration branch)
         ▲            ▲           ▲
     feature/X    feature/Y    bugfix/Z    (short-lived, per-developer branches)

                               hotfix/critical-bug
                               ▲         │
                         from main     merge back → main + develop
```

### Branch Roles

| Branch | Purpose | Who creates | Direct push? | Deleted after? |
|--------|---------|-------------|-------------|----------------|
| `main` | Production-ready code | Repo admin (once) | ❌ Never | No |
| `develop` | Integration of completed features | Repo admin (once) | ❌ Never | No |
| `release/*` | Release stabilisation | Release lead | ❌ Via PR only | ✅ After merge |
| `feature/*` | New features | Any developer | ✅ Own branch only | ✅ After PR merge |
| `bugfix/*` | Non-critical bug fixes | Any developer | ✅ Own branch only | ✅ After PR merge |
| `hotfix/*` | Critical production fixes | Senior dev / lead | ✅ Own branch only | ✅ After PR merge |
| `chore/*` | Tooling, CI, docs, deps | Any developer | ✅ Own branch only | ✅ After PR merge |

---

## 3. Branch Naming Conventions

Use **kebab-case** with a type prefix. Always be descriptive.

### Patterns

```
feature/<ticket-id>-<short-description>
bugfix/<ticket-id>-<short-description>
hotfix/<ticket-id>-<short-description>
release/<version>
chore/<short-description>
```

### Examples

```bash
# Good ✅
feature/RE-142-property-search-filters
feature/RE-200-auction-live-bidding
bugfix/RE-301-mortgage-calc-rounding-error
hotfix/RE-400-listing-image-upload-crash
release/2.3.0
chore/update-dotnet-packages
chore/add-eslint-config

# Bad ❌
my-branch
jeevan-work
fix
test123
feature/stuff
```

### Rules
- **Always include a ticket ID** (Jira, GitHub Issue, Azure DevOps work item) where applicable
- No uppercase letters in branch names
- No spaces — use hyphens
- Keep the description short but meaningful (3–6 words)
- Never name a branch `temp`, `wip`, or `test` as a final branch

---

## 4. Workflow: Day-to-Day Development

### Starting a new task

```bash
# 1. Make sure you're on develop and it's up to date
git checkout develop
git pull origin develop

# 2. Create your feature branch
git checkout -b feature/RE-142-property-search-filters

# 3. Work, commit often (small logical commits)
git add .
git commit -m "feat(search): add price range filter component"
git commit -m "feat(search): wire filter to GraphQL query"
git commit -m "test(search): add unit tests for filter logic"

# 4. Keep your branch up to date (rebase, don't merge)
git fetch origin
git rebase origin/develop

# 5. Push to remote
git push origin feature/RE-142-property-search-filters

# 6. Open a Pull Request (see Section 9)
```

### ⚠️ Golden Rules
- **Never** run `git push origin main` or `git push origin develop` directly
- **Always** rebase onto `develop` before opening a PR (keeps history linear)
- **Never** merge `main` into your feature branch — use rebase
- **Commit small and often** on your feature branch; the PR will be squashed anyway

---

## 5. Feature Branches

Feature branches are the primary unit of work. Every developer works in their own feature branch.

### Lifecycle

```
1. Branch off:   develop → feature/RE-xxx-description
2. Develop:      commit freely on your branch
3. Keep fresh:   git rebase origin/develop regularly
4. Open PR:      feature/* → develop (squash merge)
5. Review:       at least 1 reviewer required (2 recommended for complex features)
6. Merge:        squash and merge via GitHub UI
7. Delete:       branch auto-deleted after merge (GitHub setting)
```

### When to use feature branches vs. sub-features

For large features spanning multiple sprints, use **feature flags** in code rather than long-lived branches. Keep branches short-lived (ideally merged within 1–3 days). Long-lived branches = merge conflicts = pain.

```csharp
// Example: use a feature flag rather than a long branch
if (_featureFlags.IsEnabled("LiveAuctionBidding"))
{
    // New code path
}
```

---

## 6. Release Branches

When `develop` has enough features for a release, create a release branch to stabilise.

### Lifecycle

```bash
# 1. Create release branch from develop
git checkout develop
git pull origin develop
git checkout -b release/2.3.0
git push origin release/2.3.0

# 2. Only bug fixes go here — NO new features
#    Fix bugs via: bugfix/* → release/2.3.0 PR

# 3. When stable, open PR: release/2.3.0 → main
#    (squash merge)

# 4. Tag the release on main (see Section 8)
git checkout main
git pull origin main
git tag -a v2.3.0 -m "Release v2.3.0 - Property Search + Auction MVP"
git push origin v2.3.0

# 5. Port fixes back to develop via cherry-pick (NOT direct merge)
git checkout develop
git pull origin develop
git checkout -b chore/port-release-2.3.0-fixes-to-develop
git cherry-pick <commit-hash-from-release-branch>
# Open PR: chore/port-release-2.3.0-fixes-to-develop → develop

# 6. Delete release branch
git push origin --delete release/2.3.0
```

### Why cherry-pick instead of merge?
Merging `release/*` back into `develop` can bring release-specific version bumps or configuration that doesn't belong in `develop`. Cherry-picking gives you surgical control over exactly what goes back.

---

## 7. Hotfix Branches

For critical production bugs that can't wait for the next release cycle.

### Lifecycle

```bash
# 1. Branch off main (NOT develop — production is broken NOW)
git checkout main
git pull origin main
git checkout -b hotfix/RE-400-listing-image-upload-crash

# 2. Fix the bug, commit
git commit -m "fix(listings): handle null reference in image upload pipeline"

# 3. Push and open PR → main
git push origin hotfix/RE-400-listing-image-upload-crash

# 4. After PR is approved and merged to main:
#    - Tag the release on main
git checkout main && git pull origin main
git tag -a v2.2.1 -m "Hotfix v2.2.1 - Image upload crash fix"
git push origin v2.2.1

# 5. Cherry-pick the fix back to develop (mandatory)
git checkout develop && git pull origin develop
git checkout -b chore/port-hotfix-RE-400-to-develop
git cherry-pick <hotfix-commit-hash>
# Open PR → develop

# 6. Delete hotfix branch
git push origin --delete hotfix/RE-400-listing-image-upload-crash
```

### Hotfix rules
- Hotfix branches are created by senior developers / tech leads only
- Fix **only** the critical issue — do not bundle other changes
- Must be ported back to `develop` within 24 hours
- Notify the team in Slack/Teams immediately when a hotfix is deployed

---

## 8. Tags

Tags mark specific points in history — primarily **release versions**. We use **annotated tags** (not lightweight tags) because they store author, date, and message.

### Tag naming convention

Follow **Semantic Versioning** (SemVer):

```
v<MAJOR>.<MINOR>.<PATCH>

v1.0.0    # First stable release
v1.1.0    # New features, backward compatible
v1.1.1    # Bug fix (hotfix)
v2.0.0    # Breaking changes
```

### When to tag

| Event | Tag example |
|-------|-------------|
| New feature release | `v2.3.0` |
| Hotfix | `v2.2.1` |
| Major platform release | `v3.0.0` |
| Pre-release / beta | `v2.3.0-beta.1` |

### Tag commands

```bash
# Create an annotated tag
git tag -a v2.3.0 -m "Release v2.3.0 - Property search filters + mortgage calculator"

# Push a specific tag
git push origin v2.3.0

# Push all tags
git push origin --tags

# List all tags
git tag -l

# View tag details
git show v2.3.0

# Delete a tag (use sparingly)
git tag -d v2.3.0                  # delete locally
git push origin --delete v2.3.0    # delete from remote
```

### Tags vs. release branches
| | Tags | Release branches |
|--|------|-----------------|
| Purpose | Mark a point in history | Stabilise code before release |
| Post-release bug fixes | ❌ Can't add commits to a tag | ✅ Commit fixes to release branch |
| Multiple concurrent versions | ✅ Lightweight reference | ✅ Active code line |
| Use case | After merging to main | During release testing |

**Rule:** Always create a release branch first → test and fix → merge to main → then tag on main.

---

## 9. Pull Requests (PRs)

PRs are the **only way** to get code into `develop` or `main`. No exceptions.

### PR Template

Every PR description must include:

```markdown
## Summary
<!-- What does this PR do? 2-3 sentences max. -->

## Changes
- [ ] List of specific changes made
- [ ] ...

## Ticket
<!-- Link: https://jira.company.com/browse/RE-142 -->

## Testing
- [ ] Unit tests added/updated
- [ ] Manual testing done locally
- [ ] No console errors

## Screenshots (if UI change)
<!-- Before / After screenshots -->

## Checklist
- [ ] Code follows project style guide
- [ ] No sensitive data (API keys, passwords) committed
- [ ] Branch is rebased on latest develop
- [ ] PR is squash-merge ready (clean single description)
```

### PR Rules

| Rule | Detail |
|------|--------|
| **Minimum reviewers** | 1 required, 2 recommended for complex changes |
| **Branch must be up to date** | Rebase on `develop` before requesting review |
| **CI must pass** | All automated checks must be green |
| **No self-merging** | You cannot merge your own PR (unless sole developer) |
| **PR size** | Keep PRs small (<400 lines changed); split large features into multiple PRs |
| **Review turnaround** | Reviewers should respond within 1 business day |
| **No force-push after review** | Once reviewers have approved, do not force-push |

### How to open a PR (GitHub)

```bash
# After pushing your branch
gh pr create \
  --base develop \
  --title "feat(search): add property price range filters [RE-142]" \
  --body "$(cat .github/PULL_REQUEST_TEMPLATE.md)"

# Or via GitHub UI:
# Repo → Pull requests → New pull request → base: develop ← compare: feature/RE-142-...
```

### Reviewer responsibilities

- Read the **code**, not just the diff
- Check for: correctness, security, performance, test coverage, readability
- Leave **constructive, specific** comments — not "this is wrong"
- Approve only when you're genuinely comfortable shipping it
- Use **"Request changes"** if something must be fixed before merge

---

## 10. Squash Commits & Clean History

We use **squash merge** for all PRs into `develop` and `main`. This keeps the history clean — one logical commit per feature/bugfix, regardless of how many WIP commits were made on the branch.

### Why squash?

```
# Without squash (noisy):
* fix typo
* forgot to add file
* wip
* more wip
* actually fix the bug
* code review feedback
* add test
→ 7 commits for one feature. Hard to read. Hard to revert.

# With squash (clean):
* feat(search): add property price range filters (#142)
→ 1 commit. Clear. Revertable with one command.
```

### How to squash merge (GitHub UI)

1. In the PR, click the **dropdown arrow** next to the Merge button
2. Select **"Squash and merge"**
3. Edit the commit message to match the Conventional Commits format (see Section 13)
4. Click **"Confirm squash and merge"**

### How to squash locally (before opening PR, optional)

```bash
# Squash last N commits into one
git rebase -i HEAD~N

# In the editor, keep the first commit as 'pick', change the rest to 's' (squash)
# Save, then edit the combined commit message

# Force push your branch (only before PR is reviewed)
git push origin feature/RE-142-property-search-filters --force-with-lease
```

> **`--force-with-lease` vs `--force`:** Always use `--force-with-lease`. It will fail if someone else pushed to your branch, preventing accidental overwrites.

---

## 11. Branch Protection Rules

These rules **must be configured in GitHub** to enforce the standards automatically. Setup is in [Section 15](#15-enforcement-setup-github).

### For `main`

| Protection | Setting |
|-----------|---------|
| Require pull request before merging | ✅ Enabled |
| Required approvals | 2 |
| Dismiss stale reviews on new commits | ✅ Enabled |
| Require review from code owners | ✅ Enabled (when CODEOWNERS set up) |
| Require status checks to pass | ✅ Enabled (build + tests) |
| Require branches to be up to date | ✅ Enabled |
| Restrict who can push | Admins only |
| Allow force pushes | ❌ Never |
| Allow deletions | ❌ Never |

### For `develop`

| Protection | Setting |
|-----------|---------|
| Require pull request before merging | ✅ Enabled |
| Required approvals | 1 |
| Dismiss stale reviews on new commits | ✅ Enabled |
| Require status checks to pass | ✅ Enabled (build + tests) |
| Allow force pushes | ❌ Never |
| Allow deletions | ❌ Never |

---

## 12. Deleting Stale Branches

Keeping branches around after merging creates confusion and pollutes the branch list.

### Auto-delete (GitHub setting)
Enable **"Automatically delete head branches"** in GitHub repository settings. This deletes the source branch automatically when a PR is merged.

### Manual cleanup

```bash
# List branches already merged into develop
git branch --merged develop | grep -v "^\* \|main\|develop"

# Delete merged local branches
git branch --merged develop | grep -v "^\* \|main\|develop" | xargs git branch -d

# Delete a remote branch manually
git push origin --delete feature/RE-142-property-search-filters

# Prune remote tracking references (clean up stale remote refs locally)
git remote prune origin

# Or in one step
git fetch --prune
```

### Branch hygiene rules
- Delete feature/bugfix/hotfix branches **immediately** after PR is merged
- Delete release branches after they have been merged to `main` and tagged
- Never keep branches around "just in case" — git history preserves everything
- Run `git fetch --prune` before starting new work to keep your local repo clean

---

## 13. Commit Message Standards

We follow the **Conventional Commits** specification. This enables automated changelogs and semantic versioning.

### Format

```
<type>(<scope>): <short description>

[optional body]

[optional footer: BREAKING CHANGE, Refs #ticket]
```

### Types

| Type | When to use |
|------|-------------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting (no logic change) |
| `refactor` | Code restructuring (no feature/fix) |
| `test` | Adding or fixing tests |
| `chore` | Build, tooling, CI, dependencies |
| `perf` | Performance improvement |
| `revert` | Reverting a previous commit |

### Scopes (examples for this project)

`listings`, `auctions`, `mortgage`, `search`, `auth`, `api`, `graphql`, `odata`, `ui`, `ci`, `db`

### Examples

```bash
# Good ✅
feat(auctions): add real-time bidding via GraphQL subscriptions
fix(mortgage): correct compound interest rounding in calculator
docs(api): document GraphQL pagination cursor behaviour
chore(ci): add GitHub Actions workflow for .NET build
refactor(listings): extract image upload to dedicated service
test(search): add integration tests for price range filter
perf(listings): add Redis caching for property search results
feat(auth)!: replace JWT with OAuth2 PKCE flow  # ! = breaking change

# Bad ❌
fixed stuff
WIP
update
John's changes
misc fixes
```

### Breaking changes

```bash
feat(auth)!: replace JWT with OAuth2 PKCE flow

BREAKING CHANGE: All clients must update their auth flow.
Old /api/auth/token endpoint removed. See migration guide in docs/auth-migration.md.

Refs #RE-500
```

---

## 14. Quick Reference Cheatsheet

```bash
# ─── START NEW FEATURE ────────────────────────────────────────
git checkout develop && git pull origin develop
git checkout -b feature/RE-xxx-short-description

# ─── DAILY WORKFLOW ───────────────────────────────────────────
git add -p                              # Stage changes interactively
git commit -m "feat(scope): description"
git fetch origin && git rebase origin/develop  # Keep branch fresh

# ─── OPEN PULL REQUEST ────────────────────────────────────────
git push origin feature/RE-xxx-short-description
# Then open PR on GitHub: base=develop, squash merge

# ─── HOTFIX ───────────────────────────────────────────────────
git checkout main && git pull origin main
git checkout -b hotfix/RE-xxx-critical-issue
# ... fix, commit ...
git push origin hotfix/RE-xxx-critical-issue
# PR → main, then cherry-pick → develop

# ─── RELEASE ──────────────────────────────────────────────────
git checkout develop && git pull origin develop
git checkout -b release/x.y.z
# ... fix release bugs via PR → release/x.y.z ...
# PR release/x.y.z → main (squash merge)
git checkout main && git pull origin main
git tag -a vx.y.z -m "Release vx.y.z - description"
git push origin vx.y.z

# ─── CLEAN UP ─────────────────────────────────────────────────
git fetch --prune                       # Remove stale remote refs
git branch -d feature/RE-xxx-...        # Delete merged local branch
git push origin --delete feature/RE-xxx-...  # Delete remote branch

# ─── USEFUL ALIASES ───────────────────────────────────────────
git log --oneline --graph --all         # Visual branch history
git log --oneline -15                   # Last 15 commits
git diff origin/develop...HEAD          # Changes vs develop
git stash && git stash pop              # Temporarily stash work
```

---

## 15. Enforcement Setup (GitHub)

Run these steps **once** per repository to enforce the standards automatically.

### Step 1: Branch protection (via GitHub UI)

1. Go to `Settings` → `Branches` → `Add rule`
2. Configure rules for `main` and `develop` as described in Section 11
3. Alternatively, use GitHub CLI:

```bash
# Protect main: require 2 approvals, status checks, no direct push
gh api repos/{owner}/{repo}/branches/main/protection \
  --method PUT \
  --field required_pull_request_reviews='{"required_approving_review_count":2,"dismiss_stale_reviews":true}' \
  --field enforce_admins=true \
  --field restrictions=null \
  --field required_status_checks='{"strict":true,"contexts":["build","test"]}'

# Protect develop: require 1 approval
gh api repos/{owner}/{repo}/branches/develop/protection \
  --method PUT \
  --field required_pull_request_reviews='{"required_approving_review_count":1,"dismiss_stale_reviews":true}' \
  --field enforce_admins=false \
  --field restrictions=null \
  --field required_status_checks='{"strict":true,"contexts":["build","test"]}'
```

### Step 2: Auto-delete merged branches

```
Settings → General → Pull Requests → ✅ Automatically delete head branches
```

### Step 3: Default merge strategy

```
Settings → General → Pull Requests
  ❌ Uncheck "Allow merge commits"
  ❌ Uncheck "Allow rebase merging"
  ✅ Check "Allow squash merging" ONLY
  → Set default message to "Pull request title and description"
```

This forces all merges to be squash — developers cannot accidentally use a regular merge.

### Step 4: PR Template

Create `.github/PULL_REQUEST_TEMPLATE.md` in the repo root with the template from Section 9.

### Step 5: CODEOWNERS (optional but recommended)

Create `.github/CODEOWNERS`:

```
# Default: all changes require review from a senior dev
*                   @your-org/senior-devs

# API layer: GraphQL/OData changes need API team review
/src/Api/           @your-org/api-team

# Frontend: Next.js changes need frontend review
/src/Frontend/      @your-org/frontend-team

# Infrastructure / CI changes need devops review
/.github/           @your-org/devops
/infrastructure/    @your-org/devops
```

### Step 6: Commit message linting

Add `.github/workflows/lint-pr-title.yml`:

```yaml
name: Lint PR Title
on:
  pull_request:
    types: [opened, edited, synchronize, reopened]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: amannn/action-semantic-pull-request@v5
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          types: |
            feat
            fix
            docs
            style
            refactor
            test
            chore
            perf
            revert
          scopes: |
            listings
            auctions
            mortgage
            search
            auth
            api
            graphql
            odata
            ui
            ci
            db
```

### Step 7: Git hooks (local enforcement)

Add a commit-msg hook to enforce conventional commits locally:

```bash
# Install commitlint
npm install --save-dev @commitlint/cli @commitlint/config-conventional

# Create commitlint.config.js
echo "module.exports = { extends: ['@commitlint/config-conventional'] };" > commitlint.config.js

# Install husky for git hooks
npm install --save-dev husky
npx husky install
npx husky add .husky/commit-msg 'npx --no -- commitlint --edit "$1"'
```

Or add a manual `.git/hooks/commit-msg` script to each developer's local clone.

---

## Summary: What Developers Need to Know

> Copy this to your team's onboarding doc.

**The 5 things you must remember:**

1. **Never push directly to `main` or `develop`** — you will be blocked by branch protection
2. **Always branch off `develop`** for features and bugfixes; branch off `main` only for hotfixes
3. **Name your branch** like `feature/RE-123-short-description`
4. **Open a PR** when your feature is ready; request at least 1 reviewer
5. **Squash merge** your PR — keep commit history clean

**The branch lifecycle in 30 seconds:**
```
develop → feature/RE-xxx → [PR + review] → squash merge → develop → release/x.y.z → [PR] → main → tag vx.y.z
```

---

*Last updated: 2026-03-16 | Maintained by: Tech Lead*  
*Sources: [Microsoft Azure DevOps Git Branching Guidance](https://learn.microsoft.com/en-us/azure/devops/repos/git/git-branching-guidance?view=azure-devops) · [AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/choosing-git-branch-approach/git-branching-strategies.html)*
