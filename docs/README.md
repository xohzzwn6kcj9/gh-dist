# docs

User-facing documentation for **`jira-helper`**, published alongside the wheel index.

| File | What it is |
| ---- | ---------- |
| [`command-reference.md`](./command-reference.md) | The full command tree — every noun, verb, argument, flag, and example. |

## About `command-reference.md`

`command-reference.md` is **auto-generated on every release** from the CLI itself
(`jira-helper spec --markdown`), so it always matches the latest published wheel. It is
**not hand-edited** — a release overwrites it (any manual change is clobbered). To change
it, change the tool's docstrings upstream and cut a new release.

Best read **rendered on GitHub** (this repo's web UI renders Markdown). Over the raw
Pages URL it is served verbatim (unstyled text) because `.nojekyll` makes Pages serve
files as-is.

For the always-current, machine-readable form, run the tool directly:
`jira-helper --help`, `jira-helper <noun> --help`, or `jira-helper --json spec`.
