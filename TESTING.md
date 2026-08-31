# Testing Guide

Automates the test cases discussed while building this system. One script (`testing/run-tests.sh`) drives everything through `gh` — creating branches, committing files, opening PRs, triggering workflows, and asserting on the resulting labels — so you're not hand-building test fixtures each time you touch the config.

## Prerequisites

- `gh` CLI installed and authenticated (`gh auth status`) as an account with **write/admin** access to the repo — the harness needs to act as a maintainer for several tests.
- A local clone of the repo, on a **clean working tree**, with the labeling system already installed on `main` (see `SETUP.md`).
- Run all commands from the repo root: `./testing/run-tests.sh <test>`.

Each test creates a branch named `test-labeler/<test>-<timestamp>` and an open PR, and appends the PR number to a local `.test-state` file (gitignored — add it to `.gitignore` if you don't want it showing as an untracked file). Nothing is deleted automatically until you run `cleanup`, so you can inspect labels in the GitHub UI after any test before tearing it down.

## Running tests

```bash
./testing/run-tests.sh list        # see all available tests
./testing/run-tests.sh area        # run one test
./testing/run-tests.sh all         # run the fully-automated ones back to back
./testing/run-tests.sh cleanup     # close every PR + delete every branch it created
```

## What each test checks

| Test | What it does | Pass condition |
|---|---|---|
| `area` | Touches a file under `backend-api/` | `area: backend-api` applied |
| `multi` | Touches `backend-api/` + `frontend/`, then removes the `frontend/` change | `area: multi` applied, then removed when back to 1 area |
| `root` | Touches a loose root file | `area: root` applied |
| `docker` | Adds a `Dockerfile.test` | `docker` label applied automatically |
| `size-shrink` | Adds a 300-line file, then shrinks it to 5 lines | size label moves down (not just up) |
| `size-xlfiles` | Adds 35 one-line files | `size/XL` applied despite low total line count — checks alignment with `pr.size-warning.yml`'s file-count trigger |
| `triage` | Opens a PR, runs the activity scan, then comments as the author | `needs-triage` appears, then **persists** through the self-comment |
| `draft` | Opens a **draft** PR | No `needs-triage` or tier label applied |
| `exempt` | Adds a maintainer comment + `has-conflicts` label, forces thresholds to 0 | No `stale`/`needs-decision`/`final-notice` applied despite the clock condition being met |
| `sync` | Temporarily adds a throwaway label to `labels.yml`, runs Sync Labels, checks it was created, then reverts | Confirms the sync workflow's permissions are still correct (regression check for the earlier `contents: read` bug) |

### Two tests need manual interaction

**`verify-triage-cleared <PR#>`** — pairs with `triage`. That test prints a PR number and asks you to comment on it from a *different* maintainer account, then run:
```bash
./testing/run-tests.sh verify-triage-cleared 7
```
This checks that an outside maintainer's comment clears `needs-triage` — something that genuinely requires two distinct GitHub identities and can't be faked by the script.

**`tier-fast`** — tests the 7/10/14-day progression without waiting two weeks. The activity script's thresholds are overridable via `workflow_dispatch` inputs (see `ops.pr-activity-labeler.yml`); this test calls it with near-zero day thresholds so a maintainer-commented PR jumps straight to `final-notice` in seconds. It pauses partway through so you can add that outside maintainer comment first — same two-identity requirement as above. Run it directly:
```bash
./testing/run-tests.sh tier-fast
```
You can also run the underlying override manually against any PR without the script, useful for spot-checking:
```bash
gh workflow run ops.pr-activity-labeler.yml \
  -f stale_after_days=0 -f needs_decision_after_days=0 -f final_notice_after_days=0
```
Leave the inputs blank (or just don't pass `-f`) for real 7/10/14-day behavior — the defaults are baked into `activity-labeler.js` and only change when you explicitly override them.

## Why some things aren't scripted

- **Real-time day-based expiry** — genuinely can't be tested by waiting; the threshold-override mechanism above is the practical substitute.
- **Fork PR behavior / `pull_request_target` semantics** — needs an actual external fork, not just a branch on the same repo; worth a manual one-off test if you ever expect outside contributors, but not part of the automated suite.
- **Bot-authored comments not resetting the clock** — hard to simulate without a real bot (e.g. Dependabot) commenting; keep an eye on this the first time a bot actually does comment on a PR in the live repo.

## After a test run

`gh pr view <PR#> --json labels` is the fastest way to eyeball exactly what got applied, e.g.:
```bash
gh pr view 12 --json labels -q '.labels[].name'
```
Failures print inline as the script runs; scroll up for the full PASS/FAIL summary of a given run.
