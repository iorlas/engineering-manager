# Jira extraction

Goal: `raw/jira_issues.jsonl` with the schema in `data-contract.md`, including the status changelog. Raw issues must reach disk without passing through the main chat context.

Scope JQL (Jira Query Language), per project: `project = KEY AND (created >= "START" OR resolved >= "START" OR statusCategory != Done) ORDER BY created ASC`, where START = earliest window start minus 14 days. Add `jql_extra` from config.

## Path A: REST script (preferred when env vars exist)
Check `JIRA_BASE_URL`, `JIRA_EMAIL`, `JIRA_API_TOKEN` are set (print only whether each is set, never the token). If yes, write `scripts/extract_jira.py` with `urllib.request` (stdlib), Basic auth = base64(`email:token`), then:
- `GET {base}/rest/api/3/search/jql?jql=...&fields=issuetype,priority,created,resolutiondate,status,assignee,reporter,labels,parent,summary,description&expand=changelog&maxResults=100&nextPageToken=...` loop until no `nextPageToken`. If the site answers 404/410 for that endpoint, fall back to `/rest/api/3/search` with `startAt` paging.
- The inline changelog holds up to 100 entries per issue. If `changelog.total > len(histories)`, fetch `/rest/api/3/issue/KEY/changelog` pages for that issue.
- Keep only history items where `field == "status"` for `transitions`.
- Retry 3 times with 2/4/8 s waits on 429/5xx; honour `Retry-After`.
- Search `description` and `summary` for `github.com/.../pull/N` to fill `linked_pr_hint`; do not keep full text.
- Count check: total from the first response (or a separate `approximate-count` call) vs lines written.

## Path B: Haiku subagents through the Atlassian MCP (when no token)
The main agent must not fetch issues itself. Fan out one Haiku subagent per project per quarter slice (cap at 6 in parallel), each told to:
1. Call the MCP JQL search with the slice JQL, `fields` limited to the list above, page size small enough to stay reliable, until no more pages.
2. For each issue, fetch the changelog (`expand=changelog` if supported; else the issue-changelog operation) and keep only status transitions.
3. Append one JSON line per issue to `raw/jira_slice_<project>_<n>.jsonl` using the Write tool, in batches (write every 25 issues), with exactly the schema in `data-contract.md`. Copy values, never summarise or invent.
4. Return only: file path, issues written, JQL total. Not the data.
Afterwards the main agent runs a short script that concatenates the slices into `jira_issues.jsonl`, dedupes on `key`, validates every line parses as JSON with the required keys, and compares the total against the sum of JQL totals. Re-run a slice if its count is off.

Why Haiku and files: the subagent's context absorbs the data cost, which is cheap on Haiku, and the main context only sees three small values. Why validate: a model copying rows can drop or alter fields, so check mechanically.

## If Jira cannot be reached
Set `jira.mode` to `none`, continue with GitHub only, and make the report say which sections are missing and why.
