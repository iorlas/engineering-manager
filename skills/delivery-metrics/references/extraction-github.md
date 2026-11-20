# GitHub extraction

Write one script `scripts/extract_github.py`, stdlib only, calling `gh` through `subprocess.run([...list...], capture_output=True, encoding="utf-8")`. Never `shell=True`, never `--jq`, never pipes: quoting differs between Windows cmd/PowerShell and bash, and `jq` is not installed on Windows. Parse JSON in Python.

## Field names: do not trust memory
Run `gh pr list --repo OWNER/NAME --json nosuchfield`. gh answers with the list of valid field names. Pick from that list. Typical useful fields: `number,title,author,createdAt,mergedAt,closedAt,baseRefName,headRefName,isDraft,additions,deletions,changedFiles,labels,body,commits,reviews,latestReviews,reviewRequests`.

## PRs
- List by date slices so no single query hits the 1000-result search cap. Slice = one calendar month. For each slice and repo:
  `gh pr list --repo R --state merged --search "merged:YYYY-MM-DD..YYYY-MM-DD" --limit 1000 --json <fields>`
  and again with `--state closed` for unmerged (`-is:merged` in the search).
- If a slice returns exactly 1000 rows, split it in half by date and retry.
- `commits` gives each commit's `authoredDate` and `messageBody`/`messageHeadline`. First commit = min authoredDate. Scan all commit messages for AI trailers and for `Co-Authored-By` names.
- `reviews` gives author, state, submittedAt. Drop reviews by the PR author. `first_review_at` = earliest remaining; `first_approval_at` / `last_approval_at` = earliest/latest APPROVED.
- If a heavy `--json` set makes gh time out, fetch the light fields for the list, then fetch `commits,reviews` per PR with `gh pr view N --repo R --json commits,reviews`, 4 threads (`concurrent.futures.ThreadPoolExecutor`), retrying 3 times with a 2/4/8 second wait on rate limit or 502.
- Large commit lists: gh returns only the latest 100 commits per PR. That is fine for first_commit when the PR has under 100 commits; for others, accept `commit_count >= 100` and note it in caveats.

## Deploys (follow `deploy_source` from config)
- `release`: `gh release list --repo R --limit 1000 --json tagName,publishedAt,isDraft,isPrerelease` (skip drafts).
- `deployment`: `gh api "repos/R/deployments?per_page=100" --paginate` then statuses if needed; keep `environment` matching prod/production.
- `tag`: `gh api "repos/R/tags?per_page=100" --paginate` is slow to date; prefer releases. Use only as fallback.
- `workflow`: `gh run list --repo R --workflow <deploy workflow> --status success --limit 1000 --json createdAt,headSha,conclusion`.
- `merge_proxy`: derive from merged PRs into the default branch; no extra calls.
Pass the whole `gh api` command as a list: `["gh","api",url,"--paginate"]`. `--paginate` prints concatenated JSON arrays; parse with `json.JSONDecoder().raw_decode` in a loop.

## Done-check (must print)
Per repo: `merged PRs written`, `closed-unmerged written`, `slice totals`, `deploys written`, `PRs missing first_commit_at`. Fail loudly if written != sum of slice sizes. Dedupe on `(repo, number)`.

## Rate limits
Check once with `gh api rate_limit`. A normal run (a few thousand PRs) stays far under 5000 requests/hour when using list calls. Per-PR fallback calls cost 1 request each; budget them.
