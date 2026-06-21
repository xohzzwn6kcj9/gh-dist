# CLAUDE.md — working in the `gh-dist` repo

Guidance for Claude Code when this repo is checked out. Written for a
**token-limited session** (e.g. Sonnet on a monthly quota): prefer the cheap,
exact moves below over open-ended exploration.

## What this repo is — and is NOT

`gh-dist` is a **PEP 503 "simple" package index** for the `jira-helper` tool,
served as static files over GitHub Pages. It contains **only built artifacts**:

```
simple/index.html                         # index root
simple/jira-helper/index.html             # version links (PEP 503)
simple/jira-helper/*.whl                   # the wheels
simple/jira-helper/*.whl.metadata          # per-wheel METADATA
```

- It is **NOT the `jira-helper` source.** The source lives in a separate
  (private) repo. You cannot read or edit `jira-helper`'s code from here — the
  Python only exists compiled inside the `.whl` (zip) files.
- **`simple/` is generated** by release CI (`simple503`). Never hand-edit it.
- So when a user asks you to "fix a `jira-helper` bug" from this clone, you can
  **diagnose and write a report** here, but the actual code fix happens upstream.
  Don't go looking for `.py` files to patch — there are none.

> **Token economy — do not unzip the wheels to learn how the tool behaves.**
> The wheel is a build artifact; reverse-reading it burns tokens for little gain.
> To learn the command surface, run `jira-helper --help`, `jira-helper <noun>
> --help`, or `jira-helper --json spec` (the whole command tree as JSON). To
> learn its state, run the diagnostics below. Reach into a wheel only if the
> user explicitly asks to inspect the shipped bytes.

## `jira-helper` in one paragraph

A Python ≥3.11 **CLI + importable library** for Atlassian Cloud admin work —
Jira (resolve people ↔ account-ids, model an org tree, query/cache tickets) plus
Confluence page search (`page`, since v0.8.0).
Installed from this index over anonymous HTTPS — no GitHub login/token, which is
why it works on locked-down corporate networks:

```bash
# install
uv tool install jira-helper --index https://xohzzwn6kcj9.github.io/gh-dist/simple/
# upgrade to latest
uv tool install --force jira-helper --index https://xohzzwn6kcj9.github.io/gh-dist/simple/
```

Console entry point: `jira-helper`. Runtime deps (typer, httpx, PyYAML) resolve
from `pypi.org`, so PyPI must be reachable; only `jira-helper` comes from here.

## Where everything lives on disk

All config + state sit in **one directory**, resolved by this precedence:

1. `--config <path>` flag → that file's directory
2. `$JIRA_HELPER_CONFIG` (full path to the config file)
3. **`$XDG_CONFIG_HOME/jira-helper/` if `XDG_CONFIG_HOME` is set, else `~/.config/jira-helper/`** ← the default

Files in that directory:

| File                  | What it holds                                              |
| --------------------- | --------------------------------------------------------- |
| `config.yaml`         | `base_url`, `email`, `api_token` — **`chmod 600`, secret** |
| `accounts.yaml`       | name → Atlassian account-id mappings                      |
| `org.yaml`            | the org hierarchy tree                                     |
| `rules.yaml`          | virtual-parent rules (git-ignored in source)              |
| `cache.db` (+ `-wal`/`-shm`) | local SQLite ticket cache                          |
| `sync.lock`           | `flock` file guarding single-instance `sync`              |
| `logs/jira-helper.log` (+ `.1`…`.3`) | rotating CLI log (v0.7.0+, `chmod 600`, credentials redacted) |

Env vars **override** the config file at read time (handy for CI / a throwaway
shell): `JIRA_BASE_URL`, `JIRA_EMAIL`, `JIRA_API_TOKEN`.

Find the live paths without guessing: `jira-helper config path` and
`jira-helper cache path`.

## Where logs and errors go ← read this first for any error

**`jira-helper` writes every run to a rotating log file** (since **v0.7.0**) — this
is the first thing to read for any failure. It lives in a `logs/` subdir of the
config dir (alongside `config.yaml`), by default:

```
~/.config/jira-helper/logs/jira-helper.log     # + .log.1 … .log.3, rotated (~2 MB each)
```

Path resolution follows the config dir (`--config <path>`'s dir → `$JIRA_HELPER_CONFIG`'s
dir → `$XDG_CONFIG_HOME/jira-helper/` else `~/.config/jira-helper/`, with `logs/`
appended). Override the dir with `$JIRA_HELPER_LOG_DIR` or a `log_dir` config key;
disable file logging entirely with `$JIRA_HELPER_NO_LOG=1`. **Find it without guessing:
`jira-helper logs path`.**

**What it contains + redaction.** The log captures the command line, HTTP request
traces, expected errors, and **uncaught tracebacks** (so a crash always leaves a
record). **Credentials are redacted at write time** — the API token and the
`Authorization` header are replaced with `****` and never reach the file. But the log
*does* keep the Jira domain, your email, account-ids, and JQL (they make it useful for
debugging), so it is **not automatically safe to paste publicly** — review/scrub before
sharing (see "How to report a bug" below).

**How to read it — two ways:**

```bash
# Via the CLI (preferred; v0.7.0+):
jira-helper logs path                 # where the file is
jira-helper logs status               # size, line count, time span, rotation backups
jira-helper logs show --lines 100     # tail the log
jira-helper logs show --level ERROR   # only errors (credentials already redacted)

# Plain shell fallback (works even if the CLI itself is too broken to run):
tail -n 100 "$(jira-helper logs path)"
grep -i error "$(jira-helper logs path)"
tail -f "$(jira-helper logs path)"    # follow live while reproducing
```

`-v/--verbose` additionally **streams the log to stderr** for live debugging, and
`-q/--quiet` keeps it quiet (functional since **v0.7.0**). On a **pre-0.7.0** install
there is no log file and `-v/-q` are inert — fall back to capturing stderr (below).

`jira-helper` keeps **stdout for machine data** and sends all diagnostics to
**stderr**, so you can always split them: `jira-helper issue list … >out.json 2>err.txt`.
Beyond the persisted log, errors surface in three familiar places:

### 1. Expected, "clean" errors → stderr, no traceback

User-facing errors (bad config, missing resource, Jira API rejection) print a
single line to stderr and set an exit code — **no traceback**.

- Default: `code: message` (e.g. `jira api error: HTTP 401: ...`), in red.
- With `--json`: a JSON object on **stderr** — `{"error": "<code>", "message": "<msg>"}`.

| `code`            | Meaning                                              | Exit |
| ----------------- | ---------------------------------------------------- | ---- |
| `config_error`    | config missing / unreadable / incomplete             | 1    |
| `jira_api_error`  | Jira REST returned an error (auth, perms, bad JQL…)  | 1    |
| `not_found`       | config key / account / issue doesn't exist           | 2    |
| `not_implemented` | command is still a scaffold stub                     | 2    |
| `error`           | generic expected error                               | 1    |

`jira_api_error` messages carry Jira's own text, formatted as
`HTTP <status>: <Jira errorMessages>` — so `401`/`403` = credential/permission,
`400` = bad request (e.g. malformed JQL). Exit `0` = success.

### 2. Unexpected errors → a real Python traceback on stderr

If you see a **full traceback**, that is an **unhandled bug** in `jira-helper`
(an expected error would have been the clean one-liner above). A traceback is the
strongest signal that something is worth reporting upstream — capture it verbatim.
Since v0.7.0 the same traceback is **also written (redacted) to the log file**, so
`jira-helper logs show --level ERROR` recovers it even if you lost the terminal output.

### 3. The one *persisted* error: `sync`'s `last_error`

Background `sync` failures are saved into `cache.db` (a `last_error` field in the
`meta` table). This is the closest thing to a "log on disk". Read it with:

```bash
jira-helper --json sync status
# → {"last_sync_at": ..., "watermark": ..., "locked": false, "last_error": "..."}
```

This predates the log file and is still the easiest one-shot for *sync* health; for
everything else the rotating log above is the fuller record.

## Diagnostic commands (cheap — prefer these over exploration)

Run these first; they're fast, read-only, and secret-safe.

```bash
jira-helper version            # installed version (e.g. 0.8.0) — always include in a report
jira-helper doctor             # read-only health check (see below)
jira-helper --json doctor      # same, structured — safe to paste into a report
jira-helper logs show --level ERROR   # recent errors from the log file (v0.7.0+; redacted)
jira-helper logs status        # log file size / line count / time span (v0.7.0+)
jira-helper --json sync status # last sync error / watermark / lock state
jira-helper config show        # config with the token MASKED (never use --reveal in a report)
jira-helper config verify      # live credential check
jira-helper --json spec        # full command tree, for understanding the CLI surface
```

`doctor` never raises — each probe degrades to an `error: ...` string. It reports
`config_path`, `config_exists`, config validity, `config_perms`(`_ok` = is it
`600`?), a live `/myself` credential ping (→ your display name, or an error), and
`cache_path` / `cache_exists`. It is **secret-safe** (it never prints the token),
so its `--json` output is the single most useful thing to attach to a bug report.

## How to report a `jira-helper` bug

The fix lands in the upstream (private) source repo, so your job from this clone
is to **assemble a complete, secret-free report** the maintainer can act on.

**1. Collect the bundle** (all of these are safe to share as-is):

- `jira-helper version`
- The **exact command** that failed (including global flags) + expected vs. actual.
- The **stderr**: the `{"error","message"}` line, or the **full traceback** for a bug.
- `jira-helper --json doctor`
- The recent log (v0.7.0+): `jira-helper logs show --level ERROR` (and `logs status`);
  credentials are auto-redacted, but still scrub per step 2.
- For a `sync` problem: `jira-helper --json sync status`
- OS + `python3 --version`; install method (`uv tool install … --index …`).
- Minimal reproduction steps; the `--json` variant of the command if relevant.

**2. Scrub secrets — mandatory.** This repo's issue tracker is **public**. Never
include the API token, and redact the Jira domain / email / account-ids / ticket
contents:

- Use `config show` (masked) to read config — **never `config show --reveal`**,
  which prints the raw token into the report.
- The stderr you paste can leak too: a `jira_api_error` message or a traceback
  may carry the email, the `*.atlassian.net` domain, account-ids, or ticket text.
- The **log file** auto-redacts the API token + `Authorization` header, but still
  contains the Jira domain / email / account-ids / JQL — review `logs show` output
  the same way before attaching it.
- Before sharing, grep the whole bundle for `api_token`, `JIRA_API_TOKEN`, your
  `@`-email, the `*.atlassian.net` host, and any account-ids; replace each with
  `REDACTED`.
- `doctor` / `sync status` output is token-free, but `doctor` does print your
  display name and config path — redact those if your org treats them as sensitive.

**3. File it.** This public dist repo's issue tracker is the intake point (the
source is private). Issue **body in Korean** (the maintainer's language);
keep identifiers / paths / commands / code blocks verbatim in English.

```bash
# Only if the corporate network allows authenticated GitHub + `gh` is logged in:
gh issue create --repo xohzzwn6kcj9/gh-dist \
  --title "jira-helper: <짧은 증상 요약>" \
  --body-file ./jira-helper-bug.md
```

If GitHub is blocked at work (common — it's why install uses anonymous HTTPS),
**save the report as a self-contained Markdown file** next to the clone (e.g.
`./jira-helper-bug.md`) and hand it to the maintainer out-of-band. Either way,
assemble the bundle above first — the report is what makes the bug fixable.

## Conventions for editing this repo

- This is a **GitHub repo** (`xohzzwn6kcj9/gh-dist`). User-facing GitHub text —
  issue/PR bodies, commit messages, comments — in **Korean**; this `CLAUDE.md`
  and code/identifiers/paths stay English.
- **Don't touch `simple/`** by hand — it's CI-generated; a manual edit will be
  clobbered on the next release and can corrupt the index hashes.
- Releases are tag-driven (`setuptools-scm` in the source repo): pushing a
  `vX.Y.Z` tag builds the wheel and republishes this index via GitHub Actions.
  You don't release from here.
