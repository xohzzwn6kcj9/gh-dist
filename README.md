# gh-dist

Public distribution index for **`jira-helper`** — a PEP 503 "simple" package index
served over GitHub Pages. This repo holds only built wheels and their generated
index; it is **not** the source. Don't hand-edit `simple/` — CI regenerates it on
every release.

This README is the **user guide**: what `jira-helper` is, how to install, use,
update, and uninstall it. The full, always-current command reference is the tool
itself — `jira-helper --help`, `jira-helper <noun> --help`, and
`jira-helper --json spec` (the entire command tree as JSON).

---

## What is `jira-helper`?

A terminal **CLI** and importable **Python library** (≥3.11) for admin work over
**Atlassian Cloud**:

- **People ↔ account-ids** — resolve display names to Atlassian account-ids and keep
  a local mapping (`account`).
- **Org tree** — model your organization as a hierarchy (`그룹 ▸ 부문 ▸ 실 ▸ 팀 ▸ 파트`,
  any depth) and query a whole subtree's tickets with one flag (`org`).
- **Ticket queries** — list/get Jira issues by assignee, org unit, created-date range,
  or raw JQL; walk the parent tree (`issue`).
- **Virtual parents** — layer your own conditional parent rules on top of Jira's native
  links (`rule`).
- **Local cache** — fill a SQLite cache for instant parent-tree reads (`sync` / `cache`).
- **Confluence pages** — search pages under a folder by title keyword (`page`, since v0.8.0).
- **Logs** — every run is logged to a rotating file with credentials redacted (`logs`).

It installs over plain, **anonymous HTTPS** from this index — no GitHub login or token
required, so it works on locked-down corporate networks where authenticated GitHub
access is blocked.

---

## Requirements

- **Python 3.11+**
- A **Jira Cloud** site (`https://your-domain.atlassian.net`)
- A **Jira API token** — create one at
  <https://id.atlassian.com/manage-profile/security/api-tokens> (this is *not* your
  password)
- **`pypi.org` reachable over HTTPS** — runtime deps (typer, httpx, PyYAML) resolve from
  PyPI; only `jira-helper` itself comes from this index.

---

## Install

Install with [`uv`](https://docs.astral.sh/uv/). First install `uv` if you don't have it:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Then install `jira-helper` from this index:

```bash
uv tool install jira-helper --index https://xohzzwn6kcj9.github.io/gh-dist/simple/
```

`--index` keeps PyPI as the default, so dependencies resolve from PyPI while
`jira-helper` itself comes from here. This gives you the `jira-helper` console command
on your `PATH`.

> **Pin a version** by appending `==`, e.g.
> `uv tool install 'jira-helper==0.9.0' --index https://xohzzwn6kcj9.github.io/gh-dist/simple/`.
> Available versions are listed at
> <https://xohzzwn6kcj9.github.io/gh-dist/simple/jira-helper/>.

---

## First-time setup

```bash
# 1. onboard — interactively writes ~/.config/jira-helper/config.yaml (chmod 600)
jira-helper init

# 2. confirm credentials + cache health (read-only, secret-safe)
jira-helper doctor

# 3. verify the live credential against Jira
jira-helper config verify
```

`init` asks for your Jira **base URL**, **email**, and **API token**. Prefer not to use
a config file? Set environment variables instead — they override the file at read time
and let CI or a throwaway shell run without `init`:

```bash
export JIRA_BASE_URL=https://your-domain.atlassian.net
export JIRA_EMAIL=you@example.com
export JIRA_API_TOKEN=<token>
```

---

## Usage

`jira-helper [GLOBAL FLAGS] <noun> <verb> [args] [flags]`. Global flags (e.g. `--json`)
precede the noun. `--json` is available everywhere for scripting and never changes
behavior — only output. Run `jira-helper <noun> --help` for the full grammar of any
group.

### Commands at a glance

```
jira-helper init | doctor | version          # onboarding, health, version
jira-helper spec [--json]                     # full command tree (LLM/MCP ingest)
jira-helper config  init|show|get|set|edit|verify|path
jira-helper account list|get|add|remove|search|path     # name ↔ account-id
jira-helper issue   list|roots|tree|get                 # ticket queries
jira-helper org     list|show|tree|members|create|move|rename|add-member|remove-member|delete|validate|edit|path
jira-helper rule    list|edit|add|remove|test|validate|path   # virtual-parent rules
jira-helper cache   status|get|export|clear|path        # local SQLite cache
jira-helper sync    run|status|install                  # background refresh
jira-helper logs    path|status|show|clear              # CLI log file
jira-helper page    list|get|search                     # Confluence pages
```

### Map people to account-ids (`account`)

```bash
jira-helper account search "Alice Kim"           # live lookup → candidate account-ids
jira-helper account add alice 5b10ac8d82e05b22cc # record the mapping
jira-helper account list                         # show all known mappings
```

### Query tickets (`issue`)

```bash
jira-helper issue list --assignee alice                 # one person
jira-helper issue list --assignee alice --assignee bob  # several
jira-helper issue list --created-from 2024-01-01 --created-to 2024-03-31
jira-helper issue list --jql 'project = PROJA AND statusCategory != Done'
jira-helper issue get PROJA-123
jira-helper issue tree PROJA-123                        # parent/child hierarchy
jira-helper issue roots --assignee alice                # collapse to root tickets
jira-helper --json issue list --assignee alice          # machine-readable
```

### Model your org (`org`)

A unit's members are people from the account store; querying a unit expands its **whole
subtree** to those members' tickets — the same command serves "the whole office" and
"one team".

```bash
jira-helper org create 플랫폼개발실 --type 실 --member 홍길동
jira-helper org create 서비스개발팀 --type 팀 --parent 플랫폼개발실 --member alice
jira-helper org tree                       # indented hierarchy
jira-helper org members 플랫폼개발실        # who a query will hit (subtree union)
jira-helper org validate                   # dangling parents / cycles / unresolved members

jira-helper issue list --org 플랫폼개발실   # tickets for everyone under the unit
```

### Virtual parent rules (`rule`)

Layer your own conventions on top of Jira's native parent links via a gitignored
`rules.yaml` (first-match-wins) plus one-off manual edges.

```bash
jira-helper rule add PROJA-101 PROJA-50    # PROJA-101's parent is PROJA-50
jira-helper rule edit                      # author conditional rules (rules.yaml)
jira-helper rule test PROJA-101            # which rule fires + the effective parent
jira-helper rule validate                  # schema / ambiguity / cycle checks
```

### Background sync & cache (`sync` / `cache`)

`sync` fills a local SQLite cache so parent-tree reads are instant. Point it at the org
unit(s) you care about via the `sync_orgs` config key.

```bash
jira-helper config set sync_orgs "플랫폼개발실"
jira-helper sync run                  # incremental (idempotent); --full for a clean rebuild
jira-helper sync status               # last run, watermark, lock state, last error
jira-helper sync install --launchd    # emit a launchd plist (or --cron) to schedule it

jira-helper cache status              # rows, last sync, size, schema version
jira-helper cache get PROJA-123       # one cached record
```

### Confluence pages (`page`)

Find pages in a space, under a folder, whose title contains a keyword:

```bash
jira-helper page list --space DOCS --folder 123456 --title-contains runbook
jira-helper page list --space DOCS --folder 123456 --direct   # direct children only
jira-helper page get 123456                                   # one page by id
jira-helper page search "space = DOCS AND title ~ 'runbook'"  # raw CQL escape hatch
```

### As a library

```python
import jira_helper

jira_helper.account.add("alice", "5b10ac8d82e05b22cc")
jira_helper.org.create("서비스개발팀", type="팀", members=["alice"])
issues = jira_helper.issue.list(org=["플랫폼개발실"])   # whole-subtree expansion
one = jira_helper.issue.get("PROJA-123")
pages = jira_helper.page.list(space="DOCS", folder="123456", title_contains="runbook")
```

---

## Update

Re-run the install with `--force` to upgrade to the latest published release:

```bash
uv tool install --force jira-helper --index https://xohzzwn6kcj9.github.io/gh-dist/simple/
```

For a one-word upgrade, add an alias to `~/.zshrc`:

```bash
alias jh-up='uv tool install --force jira-helper --index https://xohzzwn6kcj9.github.io/gh-dist/simple/'
```

Confirm the running version any time with `jira-helper version`.

---

## Uninstall

Remove the tool itself:

```bash
uv tool uninstall jira-helper
```

This deletes the `jira-helper` command but **leaves your config and data untouched**.
To remove those too, delete the config directory (find it first — don't guess):

```bash
jira-helper config path     # → e.g. ~/.config/jira-helper/config.yaml   (run BEFORE uninstalling)
rm -rf ~/.config/jira-helper/
```

That directory holds `config.yaml` (your **secret** API token), `accounts.yaml`,
`org.yaml`, `rules.yaml`, the SQLite cache (`cache.db*`), and `logs/` — see
[Where your data lives](#where-your-data-lives) for the full list and how the path is
resolved (it may be under `$XDG_CONFIG_HOME` or a custom `--config` location).

If you scheduled background `sync` (`sync install --launchd` / `--cron`), also remove
that launchd/cron job so it stops running after uninstall.

---

## Where your data lives

All config + state sit in **one directory**, resolved by this precedence:

1. `--config <path>` flag → that file's directory
2. `$JIRA_HELPER_CONFIG` (full path to the config file) → its directory
3. `$XDG_CONFIG_HOME/jira-helper/` if `XDG_CONFIG_HOME` is set, else `~/.config/jira-helper/` ← default

| File                          | What it holds                                          |
| ----------------------------- | ------------------------------------------------------ |
| `config.yaml`                 | `base_url`, `email`, `api_token` — **`chmod 600`, secret** |
| `accounts.yaml`               | name → Atlassian account-id mappings                   |
| `org.yaml`                    | the org hierarchy tree                                 |
| `rules.yaml`                  | virtual-parent rules                                   |
| `cache.db` (+ `-wal`/`-shm`)  | local SQLite ticket cache                              |
| `logs/jira-helper.log` (+ `.1`…`.3`) | rotating CLI log (credentials redacted)         |

Find the live paths without guessing: `jira-helper config path`,
`jira-helper cache path`, `jira-helper logs path`.

---

## Troubleshooting

Every run is logged to a rotating file (since **v0.7.0**), with the API token and
`Authorization` header redacted at write time — so a failure always leaves a trail:

```bash
jira-helper doctor                    # read-only health check (config, perms, live ping)
jira-helper logs show --level ERROR   # recent errors from the log (redacted)
jira-helper logs status               # size, line count, time span, rotation backups
jira-helper --json sync status        # last sync error / watermark / lock state
```

`-v/--verbose` streams the log to stderr live; `JIRA_HELPER_NO_LOG=1` disables file
logging; `JIRA_HELPER_LOG_DIR=/path` relocates it. Errors print a clean `code: message`
line to **stderr** (a full traceback means an unhandled bug worth reporting).

**Reporting a bug:** the source is a separate private repo, so file issues against
**this** public dist repo. Collect `jira-helper version`, the failing command,
`jira-helper --json doctor`, and `jira-helper logs show --level ERROR` into a Markdown
file — then **scrub secrets** (the log keeps the Jira domain, email, account-ids, and
JQL even though the token is redacted; never use `config show --reveal`). See
[`CLAUDE.md`](./CLAUDE.md) for the full bug-report checklist.

---

## For maintainers

The version is derived from git tags by `setuptools-scm` **in the private source repo**,
so a tag push *is* the release: GitHub Actions builds the wheel and publishes it to this
index over GitHub Pages. You don't release from this repo, and you never hand-edit
`simple/`.
