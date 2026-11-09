# PR sizing and incremental-delivery policy

## Purpose

Each pull request must be small enough for an independent reviewer to understand,
test, reject, or revert it. The limit is a delivery control: it prevents a large
feature from being hidden behind a single approval and makes evidence attributable
to one behavior.

## The 500-line executable-change limit

A PR may change **at most 500 executable lines**. Count additions and deletions
in all of the following as executable change:

- application source, tests, migrations, scripts and infrastructure manifests;
- CI workflow code and configuration that changes runtime behavior;
- contract fixtures or schemas consumed by a test or a running service.

Do not evade the limit by moving production behavior into a test fixture, a shell
script, a generated file committed to Git, or a second unreviewed repository.
Lock files and vendor/generated artifacts must be isolated and justified when they
are unavoidable.

Documentation is not executable code, but it must still be reviewable. A Docs PR
above 500 changed lines requires its description to state that it is evidence or
documentation, name the linked story, and identify the focused review questions.
It never bypasses the 500-line limit for code.

## Required PR shape

Every executable PR has exactly one primary responsibility, one target branch,
one repository, and one verification command. It includes:

1. its linked issue or story;
2. acceptance criteria that can be checked from the diff;
3. a test for the changed behavior or an explicit reason why it cannot yet exist;
4. `git diff --check` evidence;
5. a rollback or containment note appropriate to its risk; and
6. a PR description that reports the executable-line count.

The count is calculated against the PR base branch before requesting review. If
the scope grows beyond 500 lines, stop, split at a contract boundary, and create a
follow-up child branch. Do not amend or force-push reviewed commits to conceal a
scope change.

## Sequencing rules

- One repository change is one PR. Related multi-repository work is a numbered
  sequence of PRs with an integration record, not one cross-repository commit.
- A consumer does not start until its producer contract is merged and pinned.
- A migration PR is separate from the API or UI behavior that relies on it when
  combining them would exceed the limit or prevent an independent rollback.
- A code PR is promoted only after its own checks pass; promotion uses the
  documented child-to-parent flow and `cherry-pick -x` rather than permanent-
  branch merges.
- A failed check blocks the next slice. It does not justify widening the active
  PR with unrelated fixes.

## Review ledger

For each increment, Docs records: source branch, target branch, issue, exact
commit, executable-line count, verification command, reviewer decision, and
residual risk. This ledger makes the final release matrix explainable without
claiming that unmerged work is deployed.
