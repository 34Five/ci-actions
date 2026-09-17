# 34Five CI building blocks

Shared GitHub Actions steps, defined once and consumed by every 34Five repository, so
that the toolchain pin and the scan policy are one definition rather than several that
happen to agree today.

## Actions

| Action | What it does |
| --- | --- |
| `actions/secret-scan` | gitleaks over the full history of the checked-out ref, scoped to HEAD, redacted output |
| `actions/bun-check` | `bun install --frozen-lockfile`, typecheck, and the test suite N times |

Each action checks out the repository itself, so the caller does not.

## Usage

```yaml
jobs:
  check:
    name: Check (typecheck + test)      # the job name is YOURS — see below
    runs-on: ubuntu-24.04
    timeout-minutes: 20
    steps:
      - uses: 34Five/ci-actions/actions/bun-check@v1

  secrets:
    name: Secrets (gitleaks)
    runs-on: ubuntu-24.04
    timeout-minutes: 20
    steps:
      - uses: 34Five/ci-actions/actions/secret-scan@v1
```

### Inputs

`bun-check` takes `bun-version` (default `1.3.11`) and `test-passes` (default `3`).
`secret-scan` takes `gitleaks-version` (default `8.30.1`). All are pinned rather than
floating, so an upstream release cannot change a build or a verdict without a commit
saying so.

`test-passes` defaults to 3 because the Cloudflare Workers suites share one D1/DO
storage across files and run single-worker: an order-dependent leak between files
surfaces as an intermittent failure that one pass will not reliably show. A suite that
does not share storage can set this to 1.

## Composite actions, not reusable workflows

Both mechanisms work. These are composite actions because a reusable workflow **renames
the check**.

A job that calls a reusable workflow reports as `<caller-job-id> / <called-job-name>`
instead of under its own name. Measured side by side in one run:

| Called as | Check reported |
| --- | --- |
| composite action | `composite-action` |
| reusable workflow | `reusable-workflow / Secrets (gitleaks)` |

Every required status check in a branch ruleset matches on that string, and a required
check that never reports blocks every merge. Composite actions run as steps inside the
caller's job, so the job name — and the check context a ruleset matches — stays entirely
the caller's. Repositories can adopt these with no ruleset edit and no window where
merges are blocked.

The cost is that a composite action cannot set `runs-on`, `timeout-minutes`,
`concurrency`, `services` or `permissions`; those stay in each caller's workflow. That
is a real duplication, and it is the price of not renaming every check at once.

## Why this repository is public

Actions and reusable workflows in a **private** repository cannot be consumed by other
repositories in the same organization. The documented *Access → accessible from
repositories in the organization* setting does not cover it: a workflow's `GITHUB_TOKEN`
cannot read another private repository, so the reference fails to resolve.

Verified before switching: composite action by tag, by branch and by commit SHA all
failed identically at the repository level, and a reusable workflow failed at workflow
startup with zero jobs — with organization policy fully permissive and the access grant
in place. The alternative was a personal access token in every consuming repository,
which would give each one read access to every private repository in the organization.
Public was the smaller cost, since nothing here is specific to the product: two actions
that install a scanner and run a package manager.

## Versioning

Consume `@v1`. It is a moving major tag: fixes and new inputs land on it, and it is not
moved across a change that breaks an existing caller. A breaking change gets `v2`, and
repositories move when they choose to. Pin a commit SHA instead if a repository needs
certainty that nothing moves underneath it.

## Keep job names identical across repositories

`Check (typecheck + test)` and `Secrets (gitleaks)` are spelled the same way in every
consuming repository on purpose. One organization-level ruleset can then require both
contexts everywhere, and a new repository complies by being named to match rather than
by someone remembering to configure it.

## Changing an action

Every consumer is affected at once — the feature and the hazard. Treat these as shipped
infrastructure: say in the commit what breaks without the change, and verify against one
repository's branch before moving `v1`.
