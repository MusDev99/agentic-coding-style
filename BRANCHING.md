# 🌿 Branching strategy

One strategy for every hobby repo. Short-lived branches off `main`, merged only through a pull request with a Conventional Commit title, then deleted.

## 📌 Rules at a glance

| # | Rule | Enforced by |
|---|---|---|
| 1 | Default branch is **`main`** (rename `master`) | Repo settings |
| 2 | Never commit or push directly to `main` | Branch ruleset |
| 3 | Branch names use a type prefix: `feat/`, `fix/`, `chore/`, `docs/` | `branch-policy.yml` |
| 4 | Commit messages and PR titles follow **Conventional Commits** | `branch-policy.yml` (PR title) |
| 5 | Merge only by PR, **squash merge**, CI green, conversations resolved | Branch ruleset |
| 6 | One concern per branch, under ~400 changed lines | Review |
| 7 | Branch is deleted on merge; stale branches cleaned monthly | Repo setting + cleanup script |

## 🏷️ Branch names

Format: `<type>/<short-kebab-name>`, lowercase, letters, digits, `.`, `_` and `-` only.

| Prefix | Use for | Example |
|---|---|---|
| `feat/` | New feature or user-visible behaviour | `feat/expense-export` |
| `fix/` | Bug fix | `fix/mobile-overflow` |
| `chore/` | Tooling, deps, CI, refactor with no behaviour change | `chore/add-prettier` |
| `docs/` | Docs only | `docs/branching-strategy` |
| `claude/` | Agent session branches (cloud Claude Code) | `claude/project-thread-0c7ogo` |

`claude/*` is allowed because agent sessions are handed a fixed branch. It follows every other rule (PR only, Conventional title) and is deleted on merge. `ccr-*` and other ad-hoc names are not allowed.

## 📝 Conventional Commits

```
<type>(<optional scope>): <imperative summary, no trailing period>

<optional body: why, not what>

<optional footer: BREAKING CHANGE: ..., Refs #12>
```

| Type | Meaning | SemVer bump |
|---|---|---|
| `feat` | New feature | minor |
| `fix` | Bug fix | patch |
| `docs` | Docs only | none |
| `chore` | Tooling, deps, housekeeping | none |
| `refactor` | Code change, no behaviour change | none |
| `perf` | Performance | patch |
| `test` | Tests only | none |
| `ci` | CI config | none |
| `build` | Build system | none |
| `revert` | Revert a commit | depends |

Add `!` after the type (`feat!: ...`) or a `BREAKING CHANGE:` footer for breaking changes (major bump).

✅ `feat(roster): add day view for phones`
✅ `fix: keep form input after server error`
❌ `[FEAT] Add day view` (old my-portfolio style, migrate away)
❌ `update stuff`

With squash merge, **the PR title becomes the commit on `main`**, so the PR title is what CI checks. Commits inside the branch should follow the format too, but only the title is enforced.

## 🔀 Pull request flow

```
main ──●─────────────────────────●── (squash merge)
        \                       /
         feat/expense-export ──●──●
```

1. `git switch main && git pull`
2. `git switch -c feat/expense-export`
3. Commit in small steps, run the repo's gate (`npm run verify` or equivalent).
4. `git push -u origin feat/expense-export`, open a PR with a Conventional title.
5. CI green + review (self-review is fine on solo repos) → **Squash and merge**.
6. Branch auto-deletes. Locally: `git switch main && git pull && git branch -d feat/expense-export`.

Hotfix = a normal `fix/` branch. No `develop` or `release/` branches: releases are SemVer tags (`vX.Y.Z`) on `main`.

## 🔒 Branch protection (ruleset on `main`)

**Settings → Rules → Rulesets → New branch ruleset**, target: default branch.

| Setting | Value |
|---|---|
| Restrict deletions | ✅ |
| Block force pushes | ✅ |
| Require a pull request before merging | ✅, 0 approvals (solo), dismiss stale approvals, require conversation resolution |
| Allowed merge methods | Squash only |
| Require status checks to pass | ✅ `branch-policy` + the repo's CI job (e.g. `verify`) |
| Require branches to be up to date | ✅ |
| Bypass list | Empty (admins included) |

Also in **Settings → General**: disable merge commits and rebase merging, enable **Automatically delete head branches**, set the squash commit message to "Pull request title and description".

Same thing with the CLI:

```bash
REPO=MusDev99/<repo>
gh api -X PATCH repos/$REPO -f default_branch=main \
  -F allow_squash_merge=true -F allow_merge_commit=false -F allow_rebase_merge=false \
  -F delete_branch_on_merge=true \
  -f squash_merge_commit_title=PR_TITLE -f squash_merge_commit_message=PR_BODY

gh api -X POST repos/$REPO/rulesets --input - <<'JSON'
{
  "name": "protect-main",
  "target": "branch",
  "enforcement": "active",
  "conditions": { "ref_name": { "include": ["~DEFAULT_BRANCH"], "exclude": [] } },
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    { "type": "pull_request", "parameters": {
        "required_approving_review_count": 0,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": false,
        "require_last_push_approval": false,
        "required_review_thread_resolution": true,
        "allowed_merge_methods": ["squash"] } },
    { "type": "required_status_checks", "parameters": {
        "strict_required_status_checks_policy": true,
        "required_status_checks": [ { "context": "branch-policy" } ] } }
  ]
}
JSON
```

Add each repo's own CI job name to `required_status_checks` once that CI exists.

## 🔁 Renaming `master` → `main`

```bash
gh api -X POST repos/$REPO/branches/master/rename -f new_name=main
# then on every clone:
git branch -m master main && git fetch origin && git branch -u origin/main main
git remote set-head origin -a
```

GitHub retargets open PRs and redirects old `master` links. Update any CI `on: push: branches: [master]`, deploy settings (Pages, Vercel, Cloud Run triggers) and docs that name `master`.

## 🧹 Stale branch cleanup

| Branch state | Action |
|---|---|
| Merged into `main` | Delete (auto on merge going forward) |
| Unmerged, no commit in 30 days, no open PR | Open a PR or delete |
| `ccr-*` / old `claude/*` with no open PR | Delete after checking the diff is on `main` or not needed |

List remote branches already merged into `main`:

```bash
git fetch --prune origin
git branch -r --merged origin/main | grep -v -E 'origin/(main|HEAD)'
```

List unmerged branches by last commit date (review before deleting):

```bash
git for-each-ref --sort=committerdate refs/remotes/origin \
  --format='%(committerdate:short) %(refname:short)' --no-merged origin/main
```

Delete one after review: `git push origin --delete <branch>`. Run monthly; never bulk-delete without reading the list.

## 🤖 Enforcement in this repo

| File | What it does |
|---|---|
| `.github/workflows/branch-policy.yml` | Fails a PR whose branch name or title breaks the rules above. No third-party actions; copy it as-is to each repo. |
| `.github/pull_request_template.md` | Checklist: Conventional title, one concern, gates green, docs updated. |

## 🗺️ Rollout roadmap (other five repos)

Baseline from the 2026-10-26 survey. One PR per repo, each on its own `chore/branching-policy` branch.

| Phase | Repo | Today | Changes | Risk |
|---|---|---|---|---|
| 1 ✅ | agentic-coding-style | `main`, unprotected, no CI | This doc + workflow + PR template, then ruleset | 🟢 |
| 2 | tokyo-itinerary | `main`, protected, Conventional | Add `branch-policy.yml`, switch ruleset to squash-only, delete `ccr-*` | 🟢 |
| 2 | pomodoro-app | `main`, CI ✅, free-text commits | Add workflow + template, ruleset, delete stale `claude/*` | 🟢 |
| 3 | my-portfolio | `main`, `[FEAT]` commits, no CI | Move CLAUDE.md "Git" section to Conventional Commits, add workflow + ruleset | 🟡 |
| 4 | task-dashboard | **`master`**, Conventional, CI ✅ | Rename to `main`, update `ci.yml` triggers + deploy, ruleset, delete `ccr-*` | 🟡 |
| 4 | Budget-tracker | **`master`**, direct commits, no CI | Rename to `main`, add CLAUDE.md + workflow + ruleset, stop direct commits | 🟠 |

```
Week 1  ── Phase 1 ✅ agentic-coding-style (source of truth)
Week 1  ── Phase 2    tokyo-itinerary, pomodoro-app (already close)
Week 2  ── Phase 3    my-portfolio (commit-style migration)
Week 2–3 ─ Phase 4    task-dashboard, Budget-tracker (master → main + deploy check)
Week 4  ── Review     every repo: ruleset active, auto-delete on, stale branches = 0
```

**Exit check per repo:** default branch `main` · ruleset `protect-main` active · `branch-policy` required · squash only · auto-delete on · no stale branches · CLAUDE.md links to this doc.
