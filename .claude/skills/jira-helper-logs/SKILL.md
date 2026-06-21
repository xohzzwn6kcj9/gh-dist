---
name: jira-helper-logs
description: Locate, read, and collect jira-helper's CLI logs for debugging. Use when jira-helper is failing / erroring / "not working", when asked where the logs are or to find/tail/grep the log file, to debug a run, to investigate a traceback or a failed `sync`, or to assemble a redacted log bundle for a bug report. Logs live at ~/.config/jira-helper/logs/jira-helper.log (rotating) and are read via `jira-helper logs show/status/path` (v0.7.0+) or `tail`/`grep` on the file. Credentials are auto-redacted at write time, but eyeball the Jira domain / email / account-ids before sharing publicly. Read-only; never prints the API token.
---

# jira-helper logs

`jira-helper` (installed from this gh-dist index) persists **every run to a rotating
log file** since **v0.7.0**. The source is private, so the log + the read-only
diagnostics are how you debug from this clone. This skill is read-only: locate, read,
and (for a bug report) collect the log — it never modifies anything and never prints
the API token.

## 1. Locate the log

```bash
jira-helper logs path        # exact path, no guessing (v0.7.0+)
```

Default: `~/.config/jira-helper/logs/jira-helper.log` (resolution follows the config
dir — `--config`'s dir → `$JIRA_HELPER_CONFIG`'s dir → `$XDG_CONFIG_HOME/jira-helper/`
else `~/.config/jira-helper/`, with `logs/` appended; `$JIRA_HELPER_LOG_DIR` overrides).
Rotated backups are `jira-helper.log.1` … `.log.3`.

If the CLI itself is too broken to run, read the file directly:

```bash
tail -n 100 "$(jira-helper logs path)"   # or hardcode ~/.config/jira-helper/logs/jira-helper.log
```

## 2. Read recent activity / errors

```bash
jira-helper logs status               # size, line count, time span, rotation backups
jira-helper logs show --lines 100     # tail
jira-helper logs show --level ERROR   # only errors (credentials already redacted)
```

Shell fallbacks (always work):

```bash
grep -i error "$(jira-helper logs path)"
tail -f "$(jira-helper logs path)"    # follow live while reproducing the problem
```

Each line is `TIMESTAMP LEVEL logger [run-id] message`; the `run-id` correlates all
lines from one invocation. An **uncaught bug** appears as an `ERROR … uncaught
exception` line followed by a (redacted) traceback. A failed background `sync` is also
recorded in `jira-helper --json sync status` (`last_error`).

## 3. Version gate

`logs …` commands need **jira-helper ≥ 0.7.0** — check with `jira-helper version`. On an
older install there is no log file and `-v/-q` are inert; reproduce the problem and
**capture stderr** instead (`… 2>err.txt`), or upgrade:
`uv tool install --force jira-helper --index https://xohzzwn6kcj9.github.io/gh-dist/simple/`.

## 4. Collect a bug bundle

Write a self-contained, **secret-free** report (the source repo is private; this dist
repo's issue tracker is **public**). Gather into one Markdown file:

```bash
jira-helper version
jira-helper --json doctor
jira-helper --json sync status
jira-helper logs show --level ERROR
# + the exact failing command (with global flags), expected vs. actual, OS + python3 --version
```

Route any saved output to a scratch path (e.g. `~/jira-helper-bug.md`), **not** into
this repo, and never touch `simple/`.

## 5. Secret hygiene (mandatory before sharing)

Credentials are **auto-redacted at write time** (the API token and `Authorization`
header are `****` in the file) — but the log still contains the **Jira domain, your
email, account-ids, and JQL/ticket text**. Before pasting anywhere public:

- grep the bundle for `api_token`, `JIRA_API_TOKEN`, your `@`-email, the
  `*.atlassian.net` host, and account-ids; replace each with `REDACTED`.
- **Never** run `jira-helper config show --reveal` (it prints the raw token).

## 6. File the report

Follow the existing **"How to report a `jira-helper` bug"** flow in this repo's
`CLAUDE.md` (issue body in Korean; identifiers/commands verbatim in English). If the
corporate network blocks authenticated GitHub, hand the saved Markdown file to the
maintainer out-of-band.

---

**Safety:** this skill uses only read-only commands (`jira-helper logs …`, `doctor`,
`sync status`, `version`) and `tail`/`grep`/`less` on the log file. It never reads
`.env` or `config.yaml` raw and never reveals the token.
