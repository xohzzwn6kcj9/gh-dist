# jira-helper command reference

Version 0.37.0 · generated from `jira-helper spec` — do not hand-edit (regenerated on each release).

Full machine-readable form: `jira-helper --json spec`. Live per-command help: `jira-helper <command> --help`.

## Global flags

Defined once on the root command; inherited by every subcommand.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `--json` | boolean | no | no |  | Machine-readable JSON output. |
| `--config` | path | no | no |  | Override config file location. |
| `--profile` | text | no | no |  | Named credential profile / Atlassian site. |
| `--cache` | path | no | no |  | Override cache file location. |
| `--no-cache` | boolean | no | no |  | Bypass the cache; force a live API call. |
| `--quiet` / `-q` | boolean | no | no |  | Quieter logs. |
| `--verbose` / `-v` | boolean | no | no |  | More verbose logs. |
| `--color` / `--no-color` | boolean | no | no |  | Colorize output. Default: config 'color' (always\|never\|auto), else auto — on at a TTY, off when piped/--json or NO_COLOR is set. |
| `--yes` / `-y` | boolean | no | no |  | Skip confirmation prompts on destructive ops. |
| `--dry-run` | boolean | no | no |  | Show what would happen; don't apply. |

## Top-level commands

### `jira-helper doctor`

Read-only health check: config valid? creds ok? cache present? Feature: F001.

Each probe degrades to an error string instead of raising; honors --json.

**Example**

```bash
jira-helper --json doctor
```

**Example output**

```text
{"config": "ok", "config_perms_ok": true, "creds": "ok (홍길동)", "cache_exists": true, ...}
```

### `jira-helper init`

First-run onboarding wizard (alias of 'config init'). Feature: F001.

**Example**

```bash
jira-helper init
```

**Example output**

```text
~/.config/jira-helper/config.yaml
```

### `jira-helper spec [flags]`

Emit the entire command tree (commands, args, flags, types, examples) for LLM/MCP ingest.

Feature: F011. '--json' emits the full structured tree; '--markdown' renders a human command reference (the form auto-published to gh-dist on each release); plain output is a compact list of command paths. Renderer flags only — no behavior change; '--markdown' takes precedence over '--json'.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `--markdown` | boolean | no | no |  | Render the command tree as a Markdown reference. |

**Example**

```bash
jira-helper --json spec
jira-helper spec --markdown
```

**Example output**

```text
{"name": "jira-helper", "version": "0.7.0",
 "groups": [{"path": "issue", "help": "Query Jira issues (F004–F008)."}, ...],
 "commands": [{"path": "issue list", "example": "jira-helper issue list ...",
               "example_output": "[{...}]", "params": [...]}, ...]}
```

### `jira-helper version`

Print version / build info.

**Example**

```bash
jira-helper version
```

**Example output**

```text
0.7.0
```

## `account` — Resolve and store name↔account-id (F002, F003).

### `jira-helper account add <name> <account_id>`

Write/update a name→account-id mapping in the store. Feature: F003.

Upserts by name: an existing name's account-id is overwritten.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `name` | text | yes | no |  | Display name. |
| `account_id` | text | yes | no |  | Atlassian account-id. |

**Example**

```bash
jira-helper account add "홍길동" 5b10ac8d82e05b22cc7d4ef5
```

**Example output**

```text
{"name": "홍길동", "account_id": "5b10ac8d82e05b22cc7d4ef5"}
```

### `jira-helper account get <name_or_id>`

Resolve a name or account-id from the store (either direction). Feature: F002/F003.

Errors with ``not_found`` (exit 2) when the name/id is not in the store.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `name_or_id` | text | yes | no |  | A name or an Atlassian account-id. |

**Example**

```bash
jira-helper account get "홍길동"
```

**Example output**

```text
{"name": "홍길동", "account_id": "5b10ac8d82e05b22cc7d4ef5"}
```

### `jira-helper account list`

List all name↔account-id mappings. Feature: F003.

**Example**

```bash
jira-helper account list
```

**Example output**

```text
[{"name": "홍길동", "account_id": "5b10ac8d82e05b22cc7d4ef5"}, ...]
```

### `jira-helper account path`

Print the account-store file location. Feature: F003.

**Example**

```bash
jira-helper account path
```

**Example output**

```text
~/.config/jira-helper/accounts.yaml
```

### `jira-helper account remove <name_or_id>`

Delete a mapping from the store (matches name or account-id). Feature: F003.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `name_or_id` | text | yes | no |  | A name or account-id to delete. |

**Example**

```bash
jira-helper account remove "홍길동"
```

**Example output**

```text
true
```

### `jira-helper account search <name>`

OUTPUT-ONLY live lookup → profile link + candidate id(s); writes nothing. Feature: F002.

Queries the live Jira user search; record a choice afterward with `account add`.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `name` | text | yes | no |  | Name to look up live. |

**Example**

```bash
jira-helper account search "홍길동"
```

**Example output**

```text
[{"displayName": "홍길동", "accountId": "5b10ac8d82e05b22cc7d4ef5",
  "emailAddress": "hong@your-domain.atlassian.net",
  "profile": "https://your-domain.atlassian.net/jira/people/5b10ac8d82e05b22cc7d4ef5"}, ...]
```

## `cache` — Inspect/manage the local SQLite cache.

### `jira-helper cache clear [flags]`

Wipe cache (optionally per-product; use -y to skip confirmation). Feature: F010.

Prompts before deleting unless the global ``-y/--yes`` is set; ``--product`` wipes only that product's rows + sync-state. Returns the issue rows removed. A full wipe (no ``--product``) also flushes the API response memoize (``http_cache.db``), so it is the way to force-refresh cached live reads.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `--product` | text | no | no |  | Wipe only this product. |

**Example**

```bash
jira-helper -y cache clear --product jira
```

**Example output**

```text
128
```

### `jira-helper cache export [flags]`

Dump the whole cache as JSON for eyeballing / grep. Feature: F006.

``--product`` limits the dump to one product (default: all).

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `--product` | text | no | no |  | Limit to one product. |

**Example**

```bash
jira-helper --json cache export --product jira
```

**Example output**

```text
[{"key": "PROJA-123", "product": "jira", "status": "In Progress",
  "effective_parent": "PROJA-50", "source": "native", ...}, ...]
```

### `jira-helper cache get <key>`

One cached record (table/JSON). Feature: F006.

Returns the flat issue row (``raw`` left as its JSON string); errors with ``not_found`` (exit 2) when the key is not cached.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `key` | text | yes | no |  | Cached record key, e.g. PROJA-123. |

**Example**

```bash
jira-helper cache get PROJA-123
```

**Example output**

```text
{"key": "PROJA-123", "product": "jira", "summary": "Fix login", "status": "In Progress",
 "assignee_id": "5b10a2...", "effective_parent": "PROJA-50", "source": "native", ...}
```

### `jira-helper cache path`

Print the cache file location (cache.db). Feature: F010.

Honors ``--cache`` / falls back to beside the resolved config.

**Example**

```bash
jira-helper cache path
```

**Example output**

```text
~/.config/jira-helper/cache.db
```

### `jira-helper cache status`

Rows (per product), last sync, on-disk size, schema version. Feature: F010.

The ``http_cache`` block reports the separate API response memoize (``http_cache.db``): its path, row count, and on-disk size.

**Example**

```bash
jira-helper cache status
```

**Example output**

```text
{"path": "~/.config/jira-helper/cache.db", "schema_version": 2, "rows": 128,
 "per_product": {"jira": 128},
 "meta": {"jira": {"last_sync_at": "2026-06-21T09:00:00Z", "watermark": "..."}},
 "size_bytes": 90112,
 "http_cache": {"path": "~/.config/jira-helper/http_cache.db", "exists": true,
                "rows": 12, "size_bytes": 24576}}
```

## `config` — Onboarding + configuration CRUD (F001).

### `jira-helper config edit`

Open the config in $EDITOR. Feature: F001.

Not yet implemented (scaffold) — exits 2 with a structured ``not_implemented`` error.

**Example**

```bash
jira-helper config edit
```

**Example output**

```text
not_implemented: jira_helper config edit: not yet implemented (scaffold)
```

### `jira-helper config get <key> [flags]`

Read one config field (masked unless --reveal). Feature: F001.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `key` | text | yes | no |  | Config key to read, e.g. base_url. |
| `--reveal` | boolean | no | no |  | Show masked values in full. |

**Example**

```bash
jira-helper config get base_url
```

**Example output**

```text
https://your-domain.atlassian.net
```

### `jira-helper config init`

Interactive Q&A → write config, chmod 600. Feature: F001.

Prompts for base URL / email / API token, then writes the config 0o600. Refuses to overwrite an existing config unless the global ``-y/--yes`` is given.

**Example**

```bash
jira-helper config init
```

**Example output**

```text
~/.config/jira-helper/config.yaml
```

### `jira-helper config path`

Print the config file location. Feature: F001.

**Example**

```bash
jira-helper config path
```

**Example output**

```text
~/.config/jira-helper/config.yaml
```

### `jira-helper config set <key> <value>`

Set one config field non-interactively (re-chmod 600). Feature: F001.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `key` | text | yes | no |  | Config key to set. |
| `value` | text | yes | no |  | New value. |

**Example**

```bash
jira-helper config set email hong@your-domain.atlassian.net
```

**Example output**

```text
~/.config/jira-helper/config.yaml
```

### `jira-helper config show [flags]`

Print the resolved config (token masked). Feature: F001.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `--reveal` | boolean | no | no |  | Show masked secrets in full. |

**Example**

```bash
jira-helper config show
```

**Example output**

```text
base_url: https://your-domain.atlassian.net
email: you@example.com
api_token: ****
```

### `jira-helper config verify`

Live auth ping + re-assert chmod 600. Feature: F001.

Calls Jira /myself with the resolved credentials; ``--profile`` selects a named credential profile.

**Example**

```bash
jira-helper config verify
```

**Example output**

```text
{"ok": true, "base_url": "https://your-domain.atlassian.net",
 "account_id": "5b10a2...", "display_name": "홍길동", ...}
```

## `issue` — Query Jira issues (F004–F008).

### `jira-helper issue get <key>`

One issue's fields (live). Feature: F004.

Returns a single flat record (key, summary, status, issue_type, assignee, assignee_id, project, parent, created, updated).

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `key` | text | yes | no |  | Issue key, e.g. PROJA-123. |

**Example**

```bash
jira-helper issue get PROJA-123
```

**Example output**

```text
{"key": "PROJA-123", "summary": "Fix login", "status": "In Progress",
 "assignee": "홍길동", "assignee_id": "5b10a2...", ...}
```

### `jira-helper issue list [flags]`

List issues by assignee/org/date/status/JQL, sorted. Feature: F004/F005/F007/F008/F045/F046.

'--org' expands a unit's subtree (whole 실 or one 팀) to its members (F007). '--roots' collapses the result to its effective root issues (F006). '--created-from/--to' filters on the creation date; '--updated-from/--to' (F044) on last-modified — and since Jira stamps 'updated' on create, '--updated-from -1w' covers 'created or modified in the last week' in one filter. Both accept an absolute date (2026-06-18) or a bare relative expression (-1w, startOfWeek()). Every '--*-to' bound is inclusive of the whole end day, to the minute, in the querying account's timezone (a bare end date becomes '<date> 23:59'). '--status-category-changed-from/--to' (#79) filters by the last *status* transition — distinct from --updated-* (which advances on ANY edit, so link/comment housekeeping resurfaces a long-closed issue). It is CACHE-ONLY (run 'sync run' first; --no-cache/--jql/empty cache raise): the flat list also AND-combines --created/--updated-* against the cache, and --assignee/--org is optional (omitted = the whole synced product). Bounds take -4w/-30d or YYYY-MM-DD (day-granular, whole-day inclusive end). '--assigned-from/--assigned-to' (F008) filters by assignee *history* — issues the person held at some point in the window (assignee WAS IN ... DURING/AFTER/BEFORE); requires --assignee or --org, is mutually exclusive with --jql, and is live-only. Status filters (F045) refine an existing filter (they can't stand alone): '--open' = open issues only (statusCategory != Done); '--status-category' selects whole buckets (To Do / In Progress / Done; aliases todo/in-progress/done); '--status' keeps only the named status(es); '--exclude-status' drops them — '--exclude-status Cancelled' drops cancelled while KEEPING completed work (a category filter can't, since Cancelled sits in the Done bucket). '--open' and '--status-category' are the same axis (use one); any status filter is mutually exclusive with --jql and forces the live path. Project filters (F047) keep/drop by project, applied to the OUTPUT (the query stays full): '--project' keeps only the named project key(s) and '--exclude-project' drops them. For a plain list this is each ticket's own project; with '--roots' it is each result's effective ROOT — a (rule-derived) root in an excluded project is dropped even if the assigned ticket that climbed to it isn't. Both accept multiple keys, are refinements (can't stand alone), and are mutually exclusive with --jql. '--sort' (F046) orders by created / updated / key / status-category-changed with '--asc'/'--desc' (default created desc; key sorts naturally, PROJA-2 before PROJA-10; status-category-changed is cache-only, #79); it orders the flat list, the --roots set, and the --roots --verbose / tree siblings alike, and is exclusive with --jql. The default (non-JSON) view is one line per ticket (KEY — summary [status]); '--roots' collapses it to the distinct root epics, and '--roots --verbose' shows the path from each root epic down to the assigned tickets as a tree — only the relevant chain, not the root's whole subtree; assigned leaves are marked '← name'. '--json' returns the underlying structure (flat list, or the nested forest).

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `--assignee` | text | no | yes |  | Assignee name(s) or account-id(s). |
| `--org` | text | no | yes |  | Org unit name(s) — expands the subtree to its members. |
| `--jql` | text | no | no |  | Raw JQL passthrough (escape hatch). |
| `--created-from` | text | no | no |  | Created on/after (YYYY-MM-DD). |
| `--created-to` | text | no | no |  | Created on/before (inclusive; YYYY-MM-DD). |
| `--updated-from` | text | no | no |  | Updated on/after (YYYY-MM-DD or relative, e.g. -1w). |
| `--updated-to` | text | no | no |  | Updated on/before (inclusive; YYYY-MM-DD or relative). |
| `--status-category-changed-from` | text | no | no |  | Status last changed on/after (YYYY-MM-DD or relative, e.g. -4w; cache-only). |
| `--status-category-changed-to` | text | no | no |  | Status last changed on/before (inclusive; YYYY-MM-DD or relative; cache-only). |
| `--assigned-from` | text | no | no |  | Was assigned on/after (history; needs --assignee/--org). |
| `--assigned-to` | text | no | no |  | Was assigned on/before (inclusive; needs --assignee/--org). |
| `--status-category` | text | no | yes |  | Status category: To Do / In Progress / Done (aliases: todo, in-progress, done). |
| `--status` | text | no | yes |  | Keep only these exact status name(s). |
| `--exclude-status` | text | no | yes |  | Drop these status name(s), e.g. Cancelled (keeps completed work). |
| `--open` | boolean | no | no |  | Open issues only (statusCategory != Done). |
| `--project` | text | no | yes |  | Keep only these project key(s), e.g. PROJA. |
| `--exclude-project` | text | no | yes |  | Drop these project key(s). |
| `--sort` | text | no | no |  | Sort field: created, updated, key, or status-category-changed (default created; status-category-changed is cache-only). |
| `--asc` | boolean | no | no |  | Sort ascending (overrides per-field default). |
| `--desc` | boolean | no | no |  | Sort descending (overrides per-field default). |
| `--roots` | boolean | no | no |  | Collapse the result to its root issues (F006). |
| `--verbose` | boolean | no | no |  | With --roots: show the path from each root epic down to the assigned tickets. |

**Example**

```bash
jira-helper issue list --assignee 홍길동 --open --sort updated
jira-helper issue list --org 플랫폼개발실 --exclude-status 취소
jira-helper issue list --status-category-changed-from -4w --sort status-category-changed
jira-helper issue list --org 플랫폼개발실 --sort key --asc
```

**Example output**

```text
PROJA-12 — Fix login [In Progress]
PROJA-15 — Add SSO [To Do]
```

### `jira-helper issue roots [flags]`

Root issues of an assignee/org set (walk effective parents). Feature: F006/F046.

Sugar for 'issue list --roots' — takes the same refinement filters as 'issue list' (status/date/project/status-category-changed/--jql), forwarded unchanged. Reads the local cache (recursive CTE over effective_parent) when populated, else walks the live API. The default view lists each root epic on one line; add '--verbose' to show the paths down to the assigned tickets as a tree (assigned leaves marked '← name'). '--sort' (F046) orders by created / updated / key / status-category-changed with '--asc'/'--desc' (default created desc). '--json' returns the underlying structure. With '--roots', both the project filter and the cache-only '--status-category-changed-from/--to' (#79) key on each result's effective root (it needs --assignee/--org here, and can't combine with --created/--updated-*).

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `--assignee` | text | no | yes |  | Assignee name(s) or account-id(s). |
| `--org` | text | no | yes |  | Org unit name(s) — expands the subtree to its members. |
| `--jql` | text | no | no |  | Raw JQL passthrough (escape hatch). |
| `--created-from` | text | no | no |  | Created on/after (YYYY-MM-DD). |
| `--created-to` | text | no | no |  | Created on/before (inclusive; YYYY-MM-DD). |
| `--updated-from` | text | no | no |  | Updated on/after (YYYY-MM-DD or relative, e.g. -1w). |
| `--updated-to` | text | no | no |  | Updated on/before (inclusive; YYYY-MM-DD or relative). |
| `--status-category-changed-from` | text | no | no |  | Status last changed on/after (YYYY-MM-DD or relative, e.g. -4w; cache-only). |
| `--status-category-changed-to` | text | no | no |  | Status last changed on/before (inclusive; YYYY-MM-DD or relative; cache-only). |
| `--assigned-from` | text | no | no |  | Was assigned on/after (history; needs --assignee/--org). |
| `--assigned-to` | text | no | no |  | Was assigned on/before (inclusive; needs --assignee/--org). |
| `--status-category` | text | no | yes |  | Status category: To Do / In Progress / Done (aliases: todo, in-progress, done). |
| `--status` | text | no | yes |  | Keep only these exact status name(s). |
| `--exclude-status` | text | no | yes |  | Drop these status name(s), e.g. Cancelled (keeps completed work). |
| `--open` | boolean | no | no |  | Open issues only (statusCategory != Done). |
| `--project` | text | no | yes |  | Keep only these project key(s), e.g. PROJA. |
| `--exclude-project` | text | no | yes |  | Drop these project key(s). |
| `--verbose` | boolean | no | no |  | Show the path down to each assigned ticket as a tree. |
| `--sort` | text | no | no |  | Sort field: created, updated, key, or status-category-changed (default created; status-category-changed is cache-only). |
| `--asc` | boolean | no | no |  | Sort ascending (overrides per-field default). |
| `--desc` | boolean | no | no |  | Sort descending (overrides per-field default). |

**Example**

```bash
jira-helper issue roots --org 플랫폼개발실 --open --project PROJA
jira-helper issue roots --assignee alice --updated-from -1w --verbose --sort key --asc
```

**Example output**

```text
PROJA-50 — Epic: Auth revamp [In Progress]
  PROJA-12 — Fix login [In Progress]  ← 홍길동
```

### `jira-helper issue tree <key> [flags]`

Parent→child hierarchy for KEY (native + virtual rules). Feature: F006/F046.

Climbs to KEY's effective root, then expands the subtree; each node is annotated with the rule that set its effective parent. Cache-backed when populated, else a live walk. The default view is an indented tree; '--json' emits the nested dict. '--sort' (F046) orders siblings within each level by created / updated / key / status-category-changed with '--asc'/'--desc' (default created desc; --sort status-category-changed is cache-only, #79 — raises on --no-cache/unsynced KEY).

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `key` | text | yes | no |  | Issue key, e.g. PROJA-123. |
| `--sort` | text | no | no |  | Sort field: created, updated, key, or status-category-changed (default created; status-category-changed is cache-only). |
| `--asc` | boolean | no | no |  | Sort ascending (overrides per-field default). |
| `--desc` | boolean | no | no |  | Sort descending (overrides per-field default). |

**Example**

```bash
jira-helper issue tree PROJA-123 --sort key --asc
```

**Example output**

```text
PROJA-50 — Epic: Auth revamp [In Progress]
  PROJA-123 — Fix login [To Do]  (via epic-single-link)
...
```

## `logs` — Locate and read the CLI log file (F013).

### `jira-helper logs clear`

Truncate the log file + drop rotation backups (use -y to skip confirm). Feature: F013.

Prompts before clearing unless the global ``-y/--yes`` is set; returns the bytes reclaimed and the number of rotation backups removed.

**Example**

```bash
jira-helper -y logs clear
```

**Example output**

```text
{"bytes_reclaimed": 40960, "backups_removed": 1}
```

### `jira-helper logs path`

Print the log file location. Feature: F013.

Honors --config / falls back to the ``logs/`` dir beside the resolved config; override the directory with $JIRA_HELPER_LOG_DIR or the ``log_dir`` config key.

**Example**

```bash
jira-helper logs path
```

**Example output**

```text
/Users/me/.config/jira-helper/logs/jira-helper.log
```

### `jira-helper logs show [flags]`

Print the last N log lines (tail), optionally filtered by level. Feature: F013.

``--level`` (case-insensitive) is applied BEFORE the ``--lines`` cap, so ``--lines 20 --level ERROR`` returns the last 20 errors. Credentials are redacted at write time, so the output never contains the API token.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `--lines` | integer | no | no | `50` | Max lines to show (default 50). |
| `--level` | text | no | no |  | Filter by level: DEBUG/INFO/WARNING/ERROR/CRITICAL. |

**Example**

```bash
jira-helper logs show --lines 20 --level ERROR
```

**Example output**

```text
2026-06-21T09:30:01+0900 WARNING jira_helper.client [a1b2c3d4] http GET /myself -> 401
2026-06-21T09:30:05+0900 ERROR jira_helper [a1b2c3d4] uncaught exception
```

### `jira-helper logs status`

File path, size, line count, time span, rotation backups. Feature: F013.

Each field degrades (never raises) when the log file doesn't exist yet.

**Example**

```bash
jira-helper --json logs status
```

**Example output**

```text
{"path": "~/.config/jira-helper/logs/jira-helper.log", "exists": true,
 "size_bytes": 40960, "lines": 512, "oldest": "2026-06-21T08:00:00+0900",
 "newest": "2026-06-21T09:30:00+0900", "backups": 1}
```

## `org` — Manage the org hierarchy + members (F009).

### `jira-helper org add-member <unit> <member...>`

Add member(s) to a unit's own list. Feature: F009.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `unit` | text | yes | no |  | Org unit name. |
| `member` | text | yes | yes |  | Member name(s) or account-id(s). |

**Example**

```bash
jira-helper org add-member 서비스개발팀 홍길동 김철수
```

**Example output**

```text
{"name": "서비스개발팀", "type": "팀", "members": ["홍길동", "김철수"]}
```

### `jira-helper org create <unit> [flags]`

Create an org unit (optional type/parent/members). Feature: F009.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `unit` | text | yes | no |  | New unit name (unique). |
| `--type` | text | no | no |  | Free-form level label (그룹/부문/실/팀/파트). |
| `--parent` | text | no | no |  | Parent unit name (omit = root). |
| `--member` | text | no | yes |  | Seed member name(s) or account-id(s). |

**Example**

```bash
jira-helper org create 서비스개발팀 --type 팀 --parent 플랫폼개발실 --member 홍길동
```

**Example output**

```text
{"name": "서비스개발팀", "type": "팀", "parent": "플랫폼개발실", "members": ["홍길동"]}
```

### `jira-helper org delete <unit> [flags]`

Delete a unit (refuses children unless --recursive). Feature: F009.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `unit` | text | yes | no |  | Unit to delete. |
| `--recursive` | boolean | no | no |  | Also delete all descendant units. |

**Example**

```bash
jira-helper org delete 서비스개발팀
```

**Example output**

```text
True
```

### `jira-helper org edit`

Open org.yaml in $EDITOR (bulk authoring). Feature: F009.

Seeds an empty org.yaml if absent, opens it in $EDITOR/$VISUAL (else vi), then prints the file path.

**Example**

```bash
jira-helper org edit
```

**Example output**

```text
~/.config/jira-helper/org.yaml
```

### `jira-helper org list [flags]`

List all org units (name, type, parent, member count). Feature: F009.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `--type` | text | no | no |  | Filter by unit type (그룹/부문/실/팀/파트). |

**Example**

```bash
jira-helper org list --type 팀
```

**Example output**

```text
[{"name": "서비스개발팀", "type": "팀", "parent": "플랫폼개발실", "member_count": 4}, ...]
```

### `jira-helper org members <unit> [flags]`

Resolved member set a query will hit (name+id, unresolved flagged). Feature: F007.

'--direct' restricts to the unit's own members (skips the subtree union).

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `unit` | text | yes | no |  | Org unit name. |
| `--direct` | boolean | no | no |  | Own members only (skip subtree union). |

**Example**

```bash
jira-helper org members 플랫폼개발실
```

**Example output**

```text
[{"name": "홍길동", "account_id": "5b10a2...", "resolved": true},
 {"name": "미등록", "account_id": "미등록", "resolved": false}]
```

### `jira-helper org move <unit> [flags]`

Re-parent a unit's subtree (--parent <name> or --root). Feature: F009.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `unit` | text | yes | no |  | Unit to re-parent. |
| `--parent` | text | no | no |  | New parent unit name. |
| `--root` | boolean | no | no |  | Make the unit a root (no parent). |

**Example**

```bash
jira-helper org move 서비스개발팀 --parent 차세대플랫폼실
```

**Example output**

```text
{"name": "서비스개발팀", "type": "팀", "parent": "차세대플랫폼실", "members": [...]}
```

### `jira-helper org path`

Print the org-store file location. Feature: F009.

**Example**

```bash
jira-helper org path
```

**Example output**

```text
~/.config/jira-helper/org.yaml
```

### `jira-helper org remove-member <unit> <member...>`

Remove member(s) from a unit's own list. Feature: F009.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `unit` | text | yes | no |  | Org unit name. |
| `member` | text | yes | yes |  | Member(s) to remove. |

**Example**

```bash
jira-helper org remove-member 서비스개발팀 김철수
```

**Example output**

```text
{"name": "서비스개발팀", "type": "팀", "members": ["홍길동"]}
```

### `jira-helper org rename <old> <new>`

Rename a unit (children's parent pointers follow). Feature: F009.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `old` | text | yes | no |  | Current unit name. |
| `new` | text | yes | no |  | New unit name. |

**Example**

```bash
jira-helper org rename 플랫폼개발실 차세대플랫폼실
```

**Example output**

```text
{"name": "차세대플랫폼실", "type": "실", "members": [...]}
```

### `jira-helper org show <unit> [flags]`

A unit's type/parent/children + direct & effective members. Feature: F009.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `unit` | text | yes | no |  | Org unit name. |
| `--direct` | boolean | no | no |  | Own members only (skip subtree union). |

**Example**

```bash
jira-helper org show 플랫폼개발실
```

**Example output**

```text
{"name": "플랫폼개발실", "type": "실", "parent": null, "children": ["서비스개발팀"],
 "members": [{"name": "홍길동", "account_id": "5b10a2...", "resolved": true}],
 "effective_members": [...]}
```

### `jira-helper org tree [<unit>]`

Render the org as an indented hierarchy. Feature: F009.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `unit` | text | no | no |  | Root unit (omit = whole forest). |

**Example**

```bash
jira-helper org tree 플랫폼개발실
```

**Example output**

```text
{"name": "플랫폼개발실", "type": "실", "member_count": 0,
 "children": [{"name": "서비스개발팀", ...}]}
```

### `jira-helper org validate`

Check the tree (dangling/cycle/dup/empty-leaf/unresolved members). Feature: F009.

Prints nothing (empty list) when clean; otherwise one row per problem.

**Example**

```bash
jira-helper org validate
```

**Example output**

```text
[{"level": "warning", "code": "unresolved_member", "unit": "서비스개발팀",
  "message": "member '...' not in account store"}]
```

## `page` — Query Confluence pages (F012).

### `jira-helper page get <page_id>`

One page's fields by id (live, v2 API). Feature: F012.

Returns a single flat record (id, title, space, uri). A missing id surfaces as a structured 'not_found' (exit 2), not a silent null.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `page_id` | text | yes | no |  | Confluence page id, e.g. 123456. |

**Example**

```bash
jira-helper page get 123456
```

**Example output**

```text
{"id": "123456", "title": "Service runbook", "space": "789",
 "uri": "https://your-domain.atlassian.net/wiki/spaces/DOCS/pages/123456/Service+runbook"}
```

### `jira-helper page list [flags]`

List pages in a space, under a folder, whose title contains a keyword. Feature: F012.

'--folder' accepts a folder content-type id OR a parent-page id (both match the whole subtree via CQL ancestor); '--direct' narrows to immediate children. '--title-contains' is a literal, case-insensitive substring filter.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `--space` | text | yes | no |  | Confluence space key (required). |
| `--folder` | text | no | no |  | Folder/parent content id — subtree via CQL ancestor. |
| `--title-contains` | text | no | no |  | Keep pages whose title contains this keyword. |
| `--direct` | boolean | no | no |  | Direct children only (parent) instead of whole subtree. |

**Example**

```bash
jira-helper page list --space DOCS --folder 123456 --title-contains runbook
```

**Example output**

```text
[{"id": "789", "title": "Service runbook", "space": "DOCS",
  "uri": "https://your-domain.atlassian.net/wiki/spaces/DOCS/pages/789/Service+runbook"}]
```

### `jira-helper page search <cql>`

Raw CQL passthrough (escape hatch). Feature: F012.

Mirrors 'issue list --jql': the CQL is sent verbatim, with no space/folder/title-contains composition. Returns the same flat shape as 'page list'.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `cql` | text | yes | no |  | Raw CQL, e.g. 'space = DOCS AND type = page'. |

**Example**

```bash
jira-helper page search "space = DOCS AND title ~ 'runbook'"
```

**Example output**

```text
[{"id": "789", "title": "Service runbook", "space": "DOCS",
  "uri": "https://your-domain.atlassian.net/wiki/spaces/DOCS/pages/789/Service+runbook"}]
```

## `rule` — Virtual-parent rules (F006).

### `jira-helper rule add <child> <parent>`

Append a one-off manual parent edge (child → parent). Feature: F006.

Upserts by child: re-adding an existing child rewrites its parent.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `child` | text | yes | no |  | Child issue key, e.g. PROJA-101. |
| `parent` | text | yes | no |  | Parent issue key, e.g. PROJA-50. |

**Example**

```bash
jira-helper rule add PROJA-101 PROJA-50
```

**Example output**

```text
{"child": "PROJA-101", "parent": "PROJA-50"}
```

### `jira-helper rule edit`

Open rules.yaml in $EDITOR (author conditional rules). Feature: F006.

Seeds an empty rules.yaml if absent, opens it in $EDITOR/$VISUAL (else vi), then prints the file path.

**Example**

```bash
jira-helper rule edit
```

**Example output**

```text
~/.config/jira-helper/rules.yaml
```

### `jira-helper rule list`

Render parsed conditional rules + manual edges. Feature: F006.

**Example**

```bash
jira-helper rule list
```

**Example output**

```text
virtual_parent_rules:
      - epic-single-link: when project=PROJA, issue_type=Epic, linked_count=1 → parent=linked
      - bracketed-parent: when project=PROJB → parent=summary_key
    manual_parents:
      - PROJA-101 → PROJA-50

Parent strategies: `linked` (the issue the links collapse to — a single link, or a set
of links that form one native subtree, resolves to that one root epic; otherwise
unresolved) and `summary_key` (the key in the summary's leading `[KEY]` mention —
e.g. `[PROJA-1] fix` → parent PROJA-1). The `linked_count` matcher (and `linked`
collapse) count only the issue's *external* links — links to the issue's own native
children / descendants (and self-links) are excluded, so an issue is never matched on,
nor parented under, its own subtree.
```

### `jira-helper rule path`

Print the rules file location. Feature: F006.

**Example**

```bash
jira-helper rule path
```

**Example output**

```text
~/.config/jira-helper/rules.yaml
```

### `jira-helper rule remove <child>`

Drop the manual edge for a child key. Feature: F006.

Prints True if an edge was removed, False if the child had none.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `child` | text | yes | no |  | Child key whose manual edge to drop. |

**Example**

```bash
jira-helper rule remove PROJA-101
```

**Example output**

```text
True
```

### `jira-helper rule test <key>`

Dry-run: effective parent for KEY + which rule fired. Feature: F006.

Reads KEY from the cache when populated, else fetches it live; --no-cache forces the live read.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `key` | text | yes | no |  | Issue key to resolve. |

**Example**

```bash
jira-helper rule test PROJA-123
```

**Example output**

```text
{"key": "PROJA-123", "native_parent": null, "effective_parent": "PROJA-50",
     "source": "epic-single-link", "linked_keys": ["PROJA-50", "PROJA-123-CHILD"],
     "external_linked_keys": ["PROJA-50"]}

`external_linked_keys` is the link set the matcher/collapse actually used — the issue's
own native descendants (and self-links) removed; it equals `linked_keys` when none applied.
```

### `jira-helper rule validate`

Schema + cycle + ambiguity checks before traversal. Feature: F006.

Prints nothing (empty list) when clean; otherwise one row per problem.

**Example**

```bash
jira-helper rule validate
```

**Example output**

```text
[{"level": "warning", "code": "ambiguous_linked", "rule": "epic-single-link", ...}]
```

## `sync` — Background cache refresh (F010).

### `jira-helper sync install [flags]`

Render the scheduler unit and write it beside the config. Feature: F010.

--launchd (default) uses --schedule as interval seconds (default 3600); --cron uses --schedule as a crontab expression. The unit is only written — loading it (launchctl load / crontab) is the returned next_step, so the live scheduler is never touched.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `--cron` | boolean | no | no |  | Install a cron unit. |
| `--launchd` | boolean | no | no |  | Install a launchd unit (default). |
| `--schedule` | text | no | no |  | Schedule expression. |

**Example**

```bash
jira-helper sync install --launchd
```

**Example output**

```text
scheduler: launchd
path: ~/.config/jira-helper/com.jira-helper.sync.plist
unit: <?xml version="1.0" encoding="UTF-8"?>...
next_step: launchctl load ~/.config/jira-helper/com.jira-helper.sync.plist
```

### `jira-helper sync run [flags]`

Incremental, idempotent cache refresh. Feature: F010.

--full forces a complete resync (wipe the product's rows, ignore the watermark); --since <ts> overrides the stored watermark lower bound.

| name | type | required | repeatable | default | description |
| --- | --- | --- | --- | --- | --- |
| `--product` | text | no | no | `jira` | Product to refresh. |
| `--full` | boolean | no | no |  | Full resync (ignore watermark). |
| `--since` | text | no | no |  | Lower bound timestamp. |

**Example**

```bash
jira-helper sync run --product jira
```

**Example output**

```text
product: jira
synced: 42
ancestors_added: 7
recomputed: 0
watermark: 2026-06-21T03:14:00.000+0000
full: False
```

### `jira-helper sync status`

Last run time, watermark, lock state, last error. Feature: F010.

**Example**

```bash
jira-helper sync status
```

**Example output**

```text
last_sync_at: 2026-06-21T03:14:05.123456+00:00
watermark: 2026-06-21T03:14:00.000+0000
locked: False
last_error: None
```

