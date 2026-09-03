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

## Notes from the first real test pass (things that look like failures but aren't)

- **"No needs-triage applied" on `area`/`root`/`docker`/`size-*` tests** — expected. Those tests only exercise `pr.area-labeler.yml`, which never touches activity labels. `needs-triage` only appears after `ops.pr-activity-labeler.yml` runs (nightly cron, or manually/via `triage`/`tier-fast`/`exempt`, which do call it).
- **`ops.label-sync.yml` logging `⛔️ Skipping delete for 'accessibility' (inputs.skipDelete on)`** — intentional. `skip-delete: true` protects every label not listed in `labels.yml` from deletion, including pre-existing ones like `accessibility`. This is the safety net working correctly, not an error.
- **The revert commit in `sync` not removing the label from the repo** — also `skip-delete` working as intended: removing an entry from `labels.yml` never deletes the actual label; the test's own explicit `gh label delete` at the end is what actually removes it.

## Windows-specific notes

If you're running this from Git Bash / Git for Windows, `GIT_ASK_YESNO=false` is set at the top of the script to suppress an interactive "Deletion of directory 'X' failed. Should I try again?" prompt that Git for Windows can throw when the last file in a directory is removed — this would otherwise hang the script waiting for input that never comes non-interactively. If you ever see this prompt from a manual `git` command outside the script, answering `n` is safe; the file deletion itself still succeeds, only the (harmless) empty-directory cleanup fails.

## Safety note on threshold overrides

`ops.pr-activity-labeler.yml`'s threshold overrides (`stale_after_days` etc.) apply to **every open PR the scan touches**, not just one — because the underlying script always scans all open PRs in a single pass. A `pr_number` input scopes a run to a single PR, and `exempt`/`tier-fast` always pass it now. **Always include `pr_number` when overriding thresholds outside the test script**, or you risk relabeling unrelated real PRs with test thresholds — which is exactly what happened to a real PR during early testing here before this scoping existed.



`gh pr view <PR#> --json labels` is the fastest way to eyeball exactly what got applied, e.g.:
```bash
gh pr view 12 --json labels -q '.labels[].name'
```
Failures print inline as the script runs; scroll up for the full PASS/FAIL summary of a given run.
