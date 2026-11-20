# engineering-manager

Claude Code plugin. One skill so far: `delivery-metrics`.

## What it does
Run `/engineering-manager:delivery-metrics`. In about 10-15 minutes it:
1. Asks once: GitHub repos, Jira project, three checkpoint dates (default: July, September, today).
2. Pulls PR data with the `gh` CLI and Jira issue history into local files (not through the chat context).
3. Computes DORA (Dora = DevOps Research and Assessment) and PR-flow metrics with plain Python (stdlib only).
4. Writes one offline HTML report: per-metric charts across the three checkpoints, AI-adoption comparison,
   bottleneck view, QA view, optional per-engineer view.

Output goes to `./em-metrics/<date>/report.html`.

## Needs on the machine
- Claude Code
- `gh` CLI, logged in (`gh auth status`)
- Python 3.9+ (`python`, `py` or `python3`), nothing to pip-install
- Jira: either Atlassian MCP connected, or env vars `JIRA_BASE_URL`, `JIRA_EMAIL`, `JIRA_API_TOKEN`

## Install
Marketplace route (repo must be hosted, e.g. on GitHub):
```
/plugin marketplace add <owner>/engineering-manager
/plugin install engineering-manager@engineering-manager
```
Copy route (no hosting): copy the folder `skills/delivery-metrics` into
`%USERPROFILE%\.claude\skills\` (Windows) or `~/.claude/skills/` (Mac/Linux). Run it as `/delivery-metrics`.

## Status
v0.1, prompt-only: the skill tells Claude how to write its own scripts. Not yet tested on Windows or against live
GitHub/Jira. Expect the first run to need a nudge.
