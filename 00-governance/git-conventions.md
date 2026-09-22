# Git Conventions

> **Read this document before making your first commit on the project.**
> This document is our team's own working agreement. It must comply with the course's
> non-negotiable rule in [`branching-policy.md`](./branching-policy.md); where the two
> disagree, `branching-policy.md` wins.

## Branch strategy

Three permanent branches: `develop`, `qa`, `main`, matching `branching-policy.md`
exactly. **No permanent branch ever accepts a direct commit or a direct merge from
another permanent branch.** You always enter through a child branch of the branch you
are targeting, and you always leave through a Pull Request.

```
develop ← feat/[description]    ← One branch per feature/user story
        ← fix/[description]     ← One branch per bugfix
        ← chore/[description]   ← Infrastructure, docs, dependency changes

qa      ← qa-promote/[description]   ← Brings one validated commit from `develop` via cherry-pick

main    ← release/[version]     ← Brings validated commits from `qa` via cherry-pick
        ← hotfix/[description]  ← Urgent production fix, still via branch + PR, never direct
```

> **Open question for `ariel5253`, not yet resolved:** `branching-policy.md` names
> `qa/[description]` as the child-branch prefix for promoting into `qa`. Git does not
> allow a branch named `qa/anything` to coexist with a branch literally named `qa` (ref
> name collision — confirmed by testing: `git switch -c qa/x` on a repo that already has
> a branch `qa` fails with `fatal: cannot lock ref 'refs/heads/qa/x': 'refs/heads/qa'
> exists`). Until this is clarified, this team uses `qa-promote/[description]` instead,
> which is functionally identical (child branch → PR → `qa`) but does not collide.

**Rules:**
- Nobody commits directly to `develop`, `qa`, or `main`
- Every task = one branch + one Pull Request
- One branch = one task (do not mix different features)
- Branches are deleted after merge

---

## Branch naming format

```
[type]/[description-in-kebab-case]

Examples:
feat/oauth2-login
fix/schedule-overlap-calculation
chore/update-spring-dependencies
qa-promote/oauth2-login
release/1.2.0
hotfix/null-token-expiration
```

---

## Promotion between branches: cherry-pick, never merge

**`merge develop → qa` and `merge qa → main` do not exist in this model.** A user story
moves forward by being re-applied on a new branch created off the *target* branch, using
`git cherry-pick -x`.

```bash
# Story already merged into develop, validated, ready for qa
git switch qa
git pull
git switch -c qa-promote/oauth2-login
git cherry-pick -x <sha-of-the-commit-in-develop>
git push -u origin qa-promote/oauth2-login
# open PR → qa
```

**The `-x` flag is mandatory.** It writes `(cherry picked from commit <sha>)` into the
new commit's message. Since cherry-picking gives the commit a new SHA, that message is
the only trail proving what in `qa` (or `main`) came from what in `develop` (or `qa`).

---

## Release branches (`qa` → `main`)

A release is cut from `main` and filled gradually — one commit per user story already
validated in `qa` (via `git cherry-pick -x`), never a bulk merge.

```bash
git switch main
git pull
git switch -c release/1.2.0
git cherry-pick -x <sha-of-story-1-in-qa>
git cherry-pick -x <sha-of-story-2-in-qa>
git push -u origin release/1.2.0
# open PR → main, list each story and its "cherry picked from" trail
```

`hotfix/*` is the only other child of `main`: an urgent fix, still merged via PR into
`main` (never a direct commit), and afterwards re-applied to `qa` and `develop` with
`cherry-pick -x` so the branches don't drift.

---

## Commit format (Conventional Commits)

```
[type]([scope]): [lowercase description, imperative mood, no trailing period]

[optional body — explain WHY, not what]

[optional footer — issue/user story references]
```

**Types:**
| Type | When to use |
|------|-------------|
| `feat` | New functionality |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting, whitespace (no logic change) |
| `refactor` | Code refactoring without behavior change |
| `test` | Add or modify tests |
| `chore` | Tooling, dependencies, CI |
| `perf` | Performance improvement |

**Examples:**
```
feat(iam): implement JWT login

fix(scheduling): correct schedule overlap validation
Closes #42

docs(api): update actor service OpenAPI contract

chore(deps): upgrade Spring Boot to 3.2.0
```

---

## Pull Request policy

- **Size:** maximum 400 lines of code (excluding tests). If larger, split it.
- **Template:** use the template at `.github/pull_request_template.md`
- **Green CI:** merge only proceeds if all pipeline checks pass

**Reviewers, by target branch:**

| Target branch | Required approvals | Who decides this rule |
|----------------|--------------------|------------------------|
| `develop` | 1 approval, 24 business hours max | The team (this document) |
| `qa` | 1 approval, contract tests (Pact) green | The team (this document) |
| `main` | **1 approval from `ariel5253`**, mandatory | The course (`branching-policy.md`), enforced by `.github/CODEOWNERS` — not negotiable |

---

## Merge policy

- Use **Squash and Merge** for `feat/fix/chore` → `develop` and `qa-promote/*` → `qa` (keeps history clean)
- Use **Merge Commit** for `release/*` and `hotfix/*` → `main` (preserves full history)
- **Do not** use Rebase & Merge (creates confusion in shared history)
- There is no merge policy for `develop → qa` or `qa → main` because that merge never happens — see "Promotion between branches" above

---

## Tags and versioning

Follow [SemVer](https://semver.org/): `MAJOR.MINOR.PATCH`

```bash
# When releasing to production
git tag -a v1.2.0 -m "Release v1.2.0: add reports module"
git push origin v1.2.0
```
