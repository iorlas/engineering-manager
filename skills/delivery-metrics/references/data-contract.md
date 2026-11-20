# Data contract

All files UTF-8. Timestamps ISO 8601 UTC strings (`2026-09-30T12:00:00Z`); parse by replacing a trailing `Z` with `+00:00` before `datetime.fromisoformat` (older Python rejects `Z`). `.jsonl` = one JSON object per line.

## config.json
```json
{
  "run_date": "2026-11-20",
  "repos": ["owner/name"],
  "jira": {"mode": "rest|mcp|none", "base_url": "", "projects": ["KEY"], "jql_extra": ""},
  "checkpoints": [
    {"label": "July",      "end": "2026-07-31", "weeks": 8},
    {"label": "September", "end": "2026-09-30", "weeks": 8},
    {"label": "Today",     "end": "2026-11-20", "weeks": 8}
  ],
  "per_engineer": "named|anonymised|off",
  "ai_markers": {"labels": [], "branch_prefixes": [], "trailers": [], "bot_authors": []},
  "deploy_source": {"owner/name": "release|deployment|tag|workflow|merge_proxy"},
  "bots_exclude": ["dependabot[bot]", "renovate[bot]"]
}
```
Window = `(end - weeks*7 days, end]`. A PR belongs to the window containing its `merged_at`.

## raw/prs.jsonl (one per PR, merged or closed-unmerged, in range)
```
repo, number, title, author, created_at, merged_at|null, closed_at|null, base_ref, head_ref,
is_draft, additions, deletions, changed_files, labels[], body_head (first 2000 chars),
first_commit_at, commit_count, co_authors[] (parsed from Co-Authored-By trailers),
commit_msg_markers[] (AI trailers found in any commit message),
reviews: [{author, state, submitted_at}], review_requested_at|null (if available),
first_review_at, first_approval_at, last_approval_at
```
Drop reviews by the PR author before storing `reviews`, then derive the last three fields from what remains (`first_review_at` = earliest review, `first_approval_at`/`last_approval_at` = earliest/latest APPROVED; null if none). Keep also `last_review_at`. `author` and `reviews[].author` are login strings. gh's `commits` items carry `authoredDate`, `messageHeadline`, `messageBody` (verify via the field-list trick); if `messageBody` is absent, scan the headline only and note it in caveats.

## raw/deploys.jsonl
```
repo, source (release|deployment|tag|workflow|merge_proxy), at, sha|null, ref|null, success (bool|null)
```

## raw/jira_issues.jsonl
```
key, project, type, priority, created, resolved|null, status, assignee|null, reporter|null,
labels[], parent_key|null, linked_pr_hint (any GitHub PR URL or key found in issue text, else null),
transitions: [{at, from, to, by}]   # status changelog, oldest first
```

## derived/metrics.json
```json
{
  "generated_at": "...",
  "checkpoints": ["July","September","Today"],
  "metrics": {
    "<metric_id>": {
      "label": "...", "unit": "days|count|percent|per_week", "proxy": false, "note": "",
      "by_checkpoint": {"July": {"value": 4.2, "n": 38, "p75": 7.1, "p90": 12.0, "low_n": false}},
      "weekly": [{"week_start": "2026-07-06", "value": 3.9, "n": 5}]
    }
  },
  "ai": {"markers_found": true, "share_by_checkpoint": {...}, "ai_vs_non_ai": {...}},
  "stages": {"<checkpoint>": {"coding":{"median":..,"n":..}, "pickup":{}, "review":{}, "approval_to_merge":{}}},
  "jira_status_dwell": {"<checkpoint>": {"<status>": {"median_days":..,"n":..}}},
  "engineers": [{"id":"E1|name","by_checkpoint":{"Today":{"prs":0,"median_pr_total_days":0,"median_review_rounds":0,"ai_share":0,"reviews_given":0}}}],
  "(shapes of ai, stages, low_n)": "see metric-definitions.md, section 'Exact shapes left open'",
  "caveats": ["..."]
}
```
The report script reads this file only. If a metric is not computable, include it with `"value": null` and a `note` saying why.
