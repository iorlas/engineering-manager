# Windows-safe scripting rules

The skill author could not test on Windows. Follow these so scripts behave the same on Windows, macOS and Linux.

- **Python only, stdlib only.** No pandas, numpy, matplotlib, requests. Allowed: json, csv, statistics, datetime, collections, pathlib, subprocess, urllib, concurrent.futures, html, re, base64, os, sys, time, shutil, argparse.
- **Find the interpreter**: try `python`, then `py -3`, then `python3`. On Windows `python3` is often a Store stub that opens the Store; test with `--version` and check the output starts with `Python 3`.
- **Run scripts as** `<interpreter> scripts/name.py` from the run folder. No shebang reliance, no `chmod`, no `./script`.
- **No bash-isms in shell commands**: no `&&` chains reliant on bash, no `$VAR`, `export`, `grep`, `jq`, `sed`, `awk`, `head`, `tail`, `wc`, `cat`, `/tmp`. Do these inside Python. In shell tool calls keep commands single and simple.
- **Paths**: `pathlib.Path`, built with `/` operator. Never hard-code `/` or `\\` separators in strings or `C:\`. Keep the run folder short (Windows path limit): `em-metrics/<date>/`.
- **Encoding**: every `open()` uses `encoding="utf-8"` (and `newline=""` for csv; `newline="\n"` when writing jsonl). Every subprocess call uses `encoding="utf-8", errors="replace"`. Windows default is cp1252 and breaks on names and emoji.
- **Console output**: print ASCII only (no arrows, emoji, box characters); the Windows console can raise UnicodeEncodeError. Report HTML may contain Unicode because it is written with utf-8.
- **subprocess**: pass a list, `shell=False`, `capture_output=True`, `timeout=120`. Find the executable with `shutil.which("gh")` and fail with a clear message if absent. Do not build quoted strings.
- **Quoting search queries**: pass `--search` value as one list element; the list form needs no extra quotes.
- **Dates**: replace trailing `Z` with `+00:00` before `fromisoformat`. Keep everything timezone-aware UTC. Avoid `strftime("%-d")` (not portable); use `%d` or `isoformat()`.
- **Long runs**: print a progress line per repo/slice, flush output (`print(..., flush=True)`), and write partial files so a crash does not lose everything. Make scripts re-runnable: skip finished slices if the output file already exists.
- **Parallelism**: `ThreadPoolExecutor` with at most 4 workers for gh/HTTP calls. No multiprocessing (spawn semantics differ on Windows).
- **Secrets**: never print or write tokens; print only "set" or "not set".
- **Antivirus/line endings**: write plain files; do not rely on CRLF vs LF when parsing (use `.splitlines()` and `json.loads`).
- **If something fails on Windows in a way these rules do not cover**, stop, show the exact error to the user, and fix the script rather than working around it in chat.
