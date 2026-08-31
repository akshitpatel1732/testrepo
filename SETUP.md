# PR Labeling System — Setup & How It Works

## Files

```
.github/                                   # sits alongside your existing CODEOWNERS, dependabot.yml, pull_request_template.md
  labeler.yml                              # path -> area-label rules
  labels.yml                               # full label definitions (colors/descriptions)
  scripts/
    activity-labeler.js                    # 7/10/14-day clock logic
    pr-metadata-labeler.js                 # shared: area:multi / size / first-contribution (used by both workflows below)
  workflows/                               # drops in alongside your existing ci.*/ops.*/pr.* files
    pr.area-labeler.yml                    # runs on every PR open/push: area + size + first-contribution
    ops.pr-activity-labeler.yml            # daily cron + manual: stale/needs-decision/final-notice/needs-triage
    pr.labels-backfill.yml                 # manual, one-time (or re-run anytime): area/size for existing open PRs
    ops.label-sync.yml                     # manual or on push to labels.yml: creates/updates labels in the repo
```

**Size tiers vs. `pr.size-warning.yml`**: your existing size-warning workflow comments (doesn't label) when a PR exceeds 500 total lines or 30 changed files. `size/XL` here is defined to trigger on that *exact same condition*, so a PR that workflow calls out as large is never sitting in a smaller `size/*` bucket — one shared definition of "large" across both systems. The S/M/L boundaries in between are ours alone and safe to retune without touching their thresholds.

## Install order (do this once)

1. Copy all files above into your repo at the matching paths.
2. **Settings → Actions → General → Workflow permissions** → set to **"Read and write permissions"**. Without this, every workflow above will fail silently on `addLabels`/`removeLabel` calls.
3. Push to `main`. This alone triggers `ops.label-sync.yml` (it watches `labels.yml`) — check the Actions tab and confirm all labels now exist in Issues → Labels.
   - If it doesn't fire (e.g. you're not on `main` as default branch), just run it manually: Actions → Sync Labels → Run workflow.
4. Run **Actions → Backfill All PR Labels → Run workflow** once. This labels every currently-open PR with area/size/multi/first-contribution *and* the activity status (needs-triage/stale/needs-decision/final-notice) in one pass.

From here on, everything is automatic:
- New/updated PRs get area + size labels within seconds (`pr.area-labeler.yml`).
- Every night, `ops.pr-activity-labeler.yml` re-scans every open PR and updates the status label.

Note on `pull_request_target` (used by `pr.area-labeler.yml`): GitHub always runs this workflow using the version of the file on your **default branch**, never the version in the PR branch, as a security measure against malicious forks. Practically this means you can't test changes to that one file on a feature branch — they only take effect once merged to `main`.

## What triggers what

| Label | How it's set |
|---|---|
| `area: *` | File paths changed, via `labeler.yml` glob rules |
| `area: multi` | >1 area label applies |
| `size/XS`…`size/XL` | Lines changed (additions+deletions) |
| `first-contribution` | Author's first-ever PR to this repo |
| `docker`, `github_actions` | Now automated by path too (Dockerfile/compose files, `.github/workflows/**`) — your other manual labels (`abandoned`, `needs-adoption`, `has-conflicts`, `overlapping-work`) stay fully manual, since those require your judgment |
| `needs-triage` | No maintainer (write/admin permission) has commented/reviewed yet |
| `stale` | 7+ days since your last comment, no author response since |
| `needs-decision` | 10+ days, same |
| `final-notice` | 14+ days, same |

**The clock only starts once you've engaged with a PR** — a PR sitting untouched for a month with no reviewer comment gets `needs-triage`, never `stale`. The moment the author (or anyone non-maintainer) comments or pushes after your last comment, the tier label is removed automatically — no manual cleanup needed. Draft PRs and PRs already carrying `abandoned`/`needs-adoption`/`has-conflicts`/`wontfix` are skipped entirely, since you've already made the call on those.

**Nothing in this system closes, merges, or deletes a PR or file — ever.** It only adds/removes labels.

## Adjusting things later

- **Change the day thresholds**: edit the three numbers (`14`, `10`, `7`) in `activity-labeler.js`.
- **Change the cron time**: edit the `cron:` line in `pr-activity-labeler.yml` (currently 03:00 UTC daily).
- **Add a new top-level folder later**: add one block to `labeler.yml`, mirroring the existing ones. Until you do, files in a new unrecognized folder will just fall through — nothing will match, so no area label gets applied (this is the one edge case worth watching for as the repo grows; add the rule as soon as a new folder appears).
- **Rename/recolor a label**: edit `labels.yml` and push (or run Sync Labels manually).
