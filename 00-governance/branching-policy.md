# Branching Policy

> **Course rule — not negotiable.**
> This file defines what the course enforces. Your team's own working agreements live in
> [`git-conventions.md`](./git-conventions.md): adapt that file freely, as long as it complies
> with this policy. Where the two disagree, this policy wins.

## The single rule

**No permanent branch ever accepts a direct commit.
You enter through a child branch, and you leave through a Pull Request.**

Everything below is that rule applied to each repository.

---

## Documentation repository (`<abbr>-docs`)

One permanent branch: `main`.

```
main  ←──PR──  <child branch>
                1 approval · ariel5253
                merge → delete branch (optional)
```

```bash
git switch main && git pull
git switch -c docs/domain-model
# ... edit the documentation ...
git push -u origin docs/domain-model
# open PR → main, get approval, merge
```

---

## Code repositories

Three permanent branches: `develop`, `qa`, `main`.
**Each one is fed only by its own children.**

| Parent | Child branch | Example | Gate |
|---|---|---|---|
| `develop` | `feat/` `fix/` `chore/` | `feat/oauth2-login` | green CI · peer review is the team's own rule |
| `qa` | `qa/` | `qa/hu-07-loan-renewal` | contract tests (Pact) · peer review is the team's own rule |
| `main` | `release/` `hotfix/` | `release/1.2.0` | **1 approval from `ariel5253`** + CODEOWNERS + conversation resolution + dismiss stale approvals |

> If your repository already uses `dev` instead of `develop`, you may keep it — but record the
> equivalence in `git-conventions.md` and use the same name everywhere (branches, PR targets, CI).

---

## Promotion happens by re-application, never by merge

**`merge develop → qa` and `merge qa → main` do not exist in this model.**
To move a user story forward, branch off the *target* branch and re-apply the commit there.

```bash
git switch qa && git pull
git switch -c qa/hu-07-loan-renewal
git cherry-pick -x <sha-of-the-commit-in-develop>
git push -u origin qa/hu-07-loan-renewal
# open PR → qa
```

**The `-x` flag is mandatory.** Re-applying a commit gives it a new SHA, so `git merge-base` can
no longer prove that what sits in `qa` came from `develop`. `-x` writes
`(cherry picked from commit <sha>)` into the message, and that trail is the only link between the
two. Without it, in week 15 nobody can demonstrate which version of a story reached production.

---

## Release branches

A release is cut from `main` and **filled gradually** — one commit per user story that has already
been validated in `qa` (a fresh commit, or a controlled `cherry-pick -x`). When complete, it goes
to `main` through a Pull Request.

```
main ──┬───────────────────────────────────── PR ──> main
       └── release/1.2.0
             ├─ commit ← HU-07 (validated in qa)
             ├─ commit ← HU-09 (validated in qa)
             └─ commit ← HU-11 (validated in qa)
```

- Default cadence: **one release per MVP** — MVP1 (week 5), MVP2 (week 10), MVP3 (week 15).
- The release Pull Request must list the user stories it carries together with their
  `cherry picked from` trail. That list is the evidence that every commit passed through `qa`.
- `hotfix/` is the only other child of `main`: an urgent production fix, re-applied afterwards to
  `qa` and `develop` so the branches do not drift.

---

## Approvals

The teacher is a gate on `main` only — in both repositories.

| Branch | Required |
|---|---|
| `develop` | green CI (required status check) · review rule set by the team |
| `qa` | contract tests / Pact (required status check) · review rule set by the team |
| `main` | **1 approval from `ariel5253`** + CODEOWNERS + conversation resolution + dismiss stale approvals |

CI and Pact are **status checks, not approvals**: they are automatic, cost nobody any time, and are
the only objective evidence that a change passed through its stage before being re-applied.

---

## What your team decides (and must write down in `git-conventions.md`)

- How you review each other on `develop` and `qa`.
- The exact naming of child branches, as long as the parent they target is unambiguous.
- A different release cadence, if one release per MVP does not fit your project.
- Whether you delete a branch after merging — recommended, but optional.
