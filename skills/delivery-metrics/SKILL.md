---
name: delivery-metrics
description: Build an engineering-delivery and AI-adoption report from GitHub and Jira. Pulls PR and issue data with the gh CLI and Jira, computes DORA metrics (deployment frequency, lead time, change failure rate, time to restore), PR cycle-time stages, QA bounce-back and per-engineer views at three checkpoints (for example July, September, today), and writes one offline HTML report with charts and insights on where the SDLC bottlenecks are and whether AI adoption shows up in the numbers. Use whenever the user mentions DORA, delivery metrics, PR cycle time, lead time, SDLC bottlenecks, engineering dashboards, measuring AI adoption, or wants a metrics report for an engineering manager, even if they never say "skill" or "report".
---

# delivery-metrics

Goal: give an engineering manager a trustworthy, expandable report that shows (1) where delivery stands now versus two earlier checkpoints, (2) where the SDLC (software development life cycle) loses time, (3) whether AI adoption is visible in the data.

## How this skill works (read once, it explains the rules)

Raw data must never flow through your context window. Pulling thousands of PRs and issues into chat is slow, expensive and error-prone (a model copying rows corrupts them). So:

- **Scripts move and compute data.** You write small Python scripts, run them, and only read their short summaries.
- **The spec is fixed here.** Metric formulas, file schemas and checkpoint windows are defined in the reference files. Two runs on the same data must give the same numbers. Follow the references exactly instead of improvising definitions.
- **You write only the words.** Insight text comes after the numbers exist, and quotes them from `metrics.json`. Never retype a figure from memory.

This version is prompt-only: no scripts ship with the skill. The person running it may be on Windows and the skill author could not test there, so follow `references/windows-and-python.md` strictly.

## Time budget
Aim for 10-15 minutes of wall time. Run extraction in parallel (see below). Do not over-engineer: a working report with honest caveats beats a polished one that took an hour.

## Workflow

### 0. Preflight (1 min)
- Find Python: try `python --version`, then `py -3 --version`, then `python3 --version`. Use the first that is 3.9+. Stop and tell the user if none exists.
- Check `gh` with `shutil.which`-style lookup or `gh --version`; `gh auth status` must succeed. If not, tell the user to run `gh auth login` and stop.
- Create the working folder `em-metrics/<YYYY-MM-DD>/` with `raw/`, `derived/`, `scripts/` inside the current directory. Everything below lives there.

### 1. Intake (ask once, in a single message)
Ask for, and offer these defaults:
1. **GitHub repos**, as `owner/name`, one or more. (Do not guess.)
2. **Jira project key(s)** and optionally a JQL (Jira Query Language) filter. Skip Jira if the user has none; the report then drops the Jira sections and says so.
3. **Checkpoints**: three end dates. Default: end of July, end of September, today of the current year. Window length default 8 weeks, trailing. Same length for all three so they compare fairly.
4. **Per-engineer view**: named, anonymised (E1, E2...), or off. Default: anonymised.
5. **AI markers**: any team-specific signal that a PR used AI (label, branch prefix, commit trailer). Defaults are in `references/metric-definitions.md`.

Write the answers to `config.json` (schema in `references/data-contract.md`). Compute the earliest window start; extraction must cover earliest start minus 14 days to today.

### 2. Discover DORA inputs (1 min, script)
DORA needs deploys and incidents, and many teams have neither in GitHub. Check, per repo, in this order: GitHub Releases, Deployments API, tags, workflow runs named like deploy/release, else merges to the default branch. Record the choice per repo in `config.json` as `deploy_source`. Anything other than real deployments or releases is a **proxy**, and the report must label it as one. Same for incidents: look for Jira issues of type Incident/Outage or bugs with a priority/label such as Critical/Production/Hotfix; if none exist, time-to-restore is reported as "not measurable" rather than guessed.

### 3. Extract (parallel, 3-6 min)
- **GitHub**: follow `references/extraction-github.md`. One script, all repos, writes `raw/prs.jsonl` and `raw/deploys.jsonl`. Nothing goes through chat.
- **Jira**: follow `references/extraction-jira.md`. Prefer the REST script when `JIRA_BASE_URL`, `JIRA_EMAIL`, `JIRA_API_TOKEN` are set. Otherwise fan out Haiku subagents (one per date slice, in parallel) that page through the Atlassian MCP and write straight to disk.
- **Verify counts.** Every extractor ends by comparing rows on disk against the source total and printing both. A mismatch is fixed or reported, never ignored.

Run GitHub extraction and Jira extraction at the same time.

### 4. Compute (about 5 min, one script of ~150-250 lines, stdlib only)
Write `scripts/compute.py` that reads `raw/` and writes `derived/metrics.json`, following `references/metric-definitions.md` and the schema in `references/data-contract.md`. No pandas, no numpy; `json`, `csv`, `statistics`, `datetime`, `collections` are enough. Use medians and p75/p90, never means alone (cycle times are long-tailed). Every metric carries its sample size `n`. If `n` is below 5 for a window, mark the value `low_n` and do not draw conclusions from it.

Sanity-check before moving on: print (from the script itself, in Python) window PR counts, the share of PRs with no review, and the 5 longest `pr_total` values. If these look absurd (zero PRs, all times negative), fix the script instead of reporting.

### 5. Analyse and write the report (Sonnet-level work, 3 min)
Follow `references/report-spec.md`. Build `report.html` from `metrics.json` with a script (so numbers are rendered, not retyped), then add insight paragraphs. Each insight must cite the metric it comes from and its `n`. Separate three kinds of statement and label them: **Observed** (in the data), **Likely** (reasonable reading), **Unknown** (cannot tell from this data).

Hold these honesty rules; the report is going to a manager who will act on it:
- Correlation with AI use is not causation. Say so once, prominently, and word insights as "coincides with".
- If no AI markers are found, say that AI adoption is not directly measurable from this data and show checkpoint trends only. Never infer AI use from speed.
- Per-engineer data is context for a conversation, not a ranking. No leaderboards, no sorting by speed, always show `n`.
- Proxies are labelled as proxies.

### 6. Hand over
Reply with the path to `report.html`, the three most decision-relevant findings in one line each, and the caveats that matter (proxies used, missing data, low `n`). Do not paste report contents into chat.

## Reference files (read when you reach that step)
- `references/data-contract.md`: config, raw and derived file schemas.
- `references/metric-definitions.md`: exact formulas, windows, AI markers.
- `references/extraction-github.md`: gh commands, chunking, count checks.
- `references/extraction-jira.md`: REST script path and Haiku-via-MCP path.
- `references/report-spec.md`: sections, charts, insight rules.
- `references/windows-and-python.md`: cross-platform rules. Read before writing any script.
