# Metric definitions

Fixed on purpose: same data in, same numbers out. All durations in days (float, 1 decimal in report). Exclude PRs by `bots_exclude` authors from every human-flow metric (keep them in a separate count). Exclude draft-never-merged PRs. Use median, plus p75 and p90.

## DORA (DevOps Research and Assessment)

| id | definition | proxy when |
|---|---|---|
| deploy_frequency | deploys per week in window (count / weeks) | source is `merge_proxy` or `tag` |
| lead_time | median of (deploy time - first_commit_at) per merged PR; deploy time = first deploy in same repo at or after merged_at. With `merge_proxy`: merged_at - first_commit_at | merge_proxy |
| change_failure_rate | % of deploys (same repo) followed within 7 days by a failure signal: a merged PR whose title or head branch matches `revert`, `hotfix`, `rollback` (the PR that does the reverting counts, not the reverted one), or a Jira Incident created in that 7-day span in the same project. No per-change link is attempted. Not measurable if neither signal exists. | always label as proxy |
| time_to_restore | median (resolved - created) of Jira incident-type issues, or critical production bugs. Not measurable if none. | always proxy |

Report which proxy applied. Do not present a proxy as the official DORA figure.

## PR flow (the bottleneck view)

Per merged PR, four stages that add up to total cycle time:
1. `coding` = pr created_at - first_commit_at (floor 0)
2. `pickup` = first_review_at - created_at (if no review: merged_at - created_at, flagged `unreviewed`)
3. `review` = last_approval_at - first_review_at (0 if single-pass; null if unreviewed)
4. `approval_to_merge` = merged_at - last_approval_at (null if unreviewed)

Also:
- `pr_cycle_time` = merged_at - created_at; `pr_total` = merged_at - first_commit_at
- `pr_throughput` = merged PRs per week
- `pr_size` = median additions+deletions; `pct_large` = share above 500 changed lines
- `review_rounds` = count of CHANGES_REQUESTED reviews plus 1 (median)
- `unreviewed_share` = merged PRs with no review by someone other than the author
- `open_pr_age` = age now of open PRs (snapshot, today only, if extracted)

Bottleneck rule: per checkpoint, over reviewed merged PRs only, `share_of_total` for a stage = (median of that stage) / (sum of the four stage medians). The stage with the largest share is `dominant`. Report `unreviewed_share` next to it, since unreviewed PRs are excluded from stage medians. State dominant stage and its share plainly. Store `share_of_total` and `dominant` in `metrics.json`.

Edge cases:
- Reviewed but never approved (only COMMENTED reviews): `review` = last review - first review, `approval_to_merge` = merged_at - last review.
- Window end: a date `end` means the end of that day (23:59:59 UTC).
- Sort/print "5 longest" by `pr_total`.
- Open PRs are not extracted, so `open_pr_age` is out of scope in v0.1; omit it.
- Direction: each metric has `better: lower|higher` in `metrics.json`; the report's change arrow is coloured by that (deploy_frequency, throughput, ai_share: higher; times, rates, size, dwell: lower; ai_share and size are neutral, no colour).

## Jira flow and QA (only when Jira data exists)

Cohorts: `jira_cycle_time`, `qa_wait`, `bounce_back_rate`, `reopen_rate`, `status_dwell` use issues **resolved** in the window (dwell counted over all their visits). `bug_ratio` and `bug_escape` use issues **created** in the window. `wip` is a snapshot at window end. "Done" = the issue's final status when `resolved` is set, plus any status named Done/Closed/Resolved/Released. QA statuses = names containing QA/Test/Verify/UAT; "In Review" is a code-review status, not QA, and a move In Review to In Progress is not a bounce-back.

- `jira_cycle_time` = resolved - first transition into any "in progress"-like status (status names containing progress/develop/doing); fallback created.
- `status_dwell`: per status, median days an issue spends in it (sum over visits). The largest dwell outside "Backlog/To Do/Done" is the Jira bottleneck.
- `qa_wait`: median dwell in the QA statuses defined above.
- `bounce_back_rate` = % of issues with a transition from a QA/Test status back to an in-progress/dev status.
- `reopen_rate` = % of resolved issues later moved out of a Done status.
- `bug_ratio` = Bugs created / all issues created in window; `bug_escape` = bugs created in window per merged PR in window.
- `wip` = issues in progress at window end.

## AI adoption markers (deterministic detection)

A PR is **AI-assisted** if ANY of:
- any commit message contains a trailer matching the regex (case-insensitive) `Co-Authored-By:.*(claude|copilot|cursor|codex|gemini|devin|aider)` or the text `Generated with Claude Code`
- author or a reviewer login (lowercased) contains `copilot`, `devin-ai`, `claude`, or equals an entry of `ai_markers.bot_authors` (exact, case-insensitive)
- PR carries a label in `ai_markers.labels` (exact), a head branch starting with an `ai_markers.branch_prefixes` entry, or a PR body or commit message containing an `ai_markers.trailers` entry (literal, case-insensitive substring)
- PR body (`body_head`) contains `Generated with` plus Claude/Copilot/Cursor, or the emoji robot

Compute per checkpoint: `ai_share` (AI-assisted merged PRs / merged PRs), `n_ai`, `n_non_ai`.
Compare AI-assisted vs non-AI-assisted within the same checkpoint on: pr_total, pickup, review, pr_size, review_rounds, unreviewed_share. Mark `low_n` if either group has n < 5.

If `ai_share` is 0 in all checkpoints, set `markers_found: false`. The report must then say AI adoption is not directly measurable here, and list what the team could add (commit trailer convention, PR label) to make it measurable. Do not infer.

Caveat to print next to every AI comparison: AI-assisted PRs differ in kind (task type, author seniority, size), so differences coincide with AI use and do not prove it caused them.

## Weekly series
Emit weekly points (week starts Monday, across the extraction range) only for: deploy_frequency and pr_throughput (count per week; never null, zero is a value), and pr_total, pickup, ai_share (value null when the week has fewer than 3 merged PRs). No weekly series for change_failure_rate, time_to_restore or Jira metrics. Weekly points have shape `{week_start, value, n}`.

## Exact shapes left open in the contract
- `ai.share_by_checkpoint`: `{cp: {"value": share, "n_ai": int, "n_total": int, "low_n": bool}}`
- `ai.ai_vs_non_ai`: `{metric_id: {cp: {"ai": {"value","n"}, "non_ai": {"value","n"}, "low_n": bool}}}`
- `stages[cp][stage]`: `{"median", "n", "share_of_total"}`; plus `stages[cp]["dominant"]` and `stages[cp]["unreviewed_share"]`
- `low_n` is true when `n < 5`; a metric with `n = 0` has `value: null` and the summary shows "n/a"; `time_to_restore` with n < 5 is shown with its n and the low_n grey.

## Per-engineer view (if enabled)
Per author with at least 3 merged PRs in a checkpoint: `prs`, `median_pr_total_days`, `median_review_rounds`, `ai_share`, `reviews_given`. Authors with fewer PRs are folded into one row "others". Always show n. When `anonymised`, assign E1, E2... in a seeded-shuffled order (shuffle the sorted login list with `random.Random(42)`) so the number carries no rank; keep the login mapping out of the report and out of `metrics.json`; the report lists rows in id order.
