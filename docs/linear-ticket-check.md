# Linear ticket check

Reusable workflow: [`.github/workflows/linear-ticket-check.yml`](../.github/workflows/linear-ticket-check.yml)

Fails a pull request unless its **head branch name** contains a Linear issue key. The
reported check name is **`Linear ticket`**; that string is a contract, because it is what
repos put in their required-status-check list. Do not rename the job.

## Why the branch name

Linear links an issue when it finds a key in the branch name, commit subject, PR title
or PR body. Only the branch name is free of other conventions: under squash-merge the PR
title becomes the commit, which conventional commits and release-please own. Linear's
"Copy git branch name" button already produces `<user>/prd-123-title`, so linking is a
side effect of normal work. Background: design decision 0005.

## Rules

| | |
|---|---|
| Pattern | `[A-Za-z][A-Za-z0-9]+-[0-9]+`, matched anywhere in `github.head_ref` |
| Case | Insensitive. Linear keys are `PRD-123`; Linear's branch button emits `prd-123`. Both pass. |
| Team prefixes | Never hardcoded. Any prefix matches, so a new Linear team needs no change here. |
| Exempt | PRs where `github.actor` **or** the PR author is in `exempt_actors`. Default: `nx1-release-bot[bot],dependabot[bot],nx1-updatecli[bot]`. Checking both means a bot PR stays exempt when a human edits its title. |
| Permissions | None. No checkout, no API call; the check is a regex on a string GitHub already supplies. |

The generic pattern is deliberately loose (`v2-123` would pass). That is the accepted
trade-off for never causing an org-wide outage when team twelve opens its first PR.

## Wiring a repo

`.github/workflows/linear-ticket.yml` in the consuming repo:

```yaml
name: Linear ticket

on:
  pull_request:
    # `edited` re-runs the check after a branch rename or base change; `synchronize`
    # covers new pushes. Do not drop either.
    types: [opened, edited, synchronize, reopened]

permissions: {}

jobs:
  linear-ticket:
    uses: nexuscognitive/global_ci_template/.github/workflows/linear-ticket-check.yml@main
    # Optional override; comma-separated logins.
    # with:
    #   exempt_actors: 'nx1-release-bot[bot],dependabot[bot],nx1-updatecli[bot]'
```

Then, **only after the workflow has reported at least once in that repo**, add
`Linear ticket` to the branch protection / ruleset required checks. A required check
that never reports leaves every PR pending forever with no error. This is the top
failure mode of the rollout; roll out per repo, evaluate mode first.

## What a developer sees on failure

The step prints the exact fix: use Linear's "Copy git branch name" and open the PR from
that branch, or rename the current branch to include the key and push. Bot PRs are
exempt by identity, so the fix is never "add a fake key".

## Prerequisite for exemptions to work

release-please and updatecli must run as GitHub Apps. A PR authored via a personal `GH_PAT`
is authored by that person and cannot be exempted by identity (PRD-1142).

## Where this lives

In `global_ci_template` for the pilot. A verbatim copy sits in `engineering-kit`
(`.github/workflows/linear-ticket-check.yml`) and becomes the canonical one when the kit
becomes a repository; callers then switch the `uses:` path.
