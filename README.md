# gh-dist

Public distribution index for **`jira-helper`** — a PEP 503 "simple" package index
served over GitHub Pages. This repo holds only built wheels and their generated
index; it is **not** the source. Don't hand-edit `simple/` — CI regenerates it on
every release.

## Install / update

Install with [`uv`](https://docs.astral.sh/uv/) over anonymous HTTPS — no GitHub
login or token required (works on networks where authenticated GitHub access is
blocked):

```bash
uv tool install jira-helper --index https://xohzzwn6kcj9.github.io/gh-dist/simple/
```

Upgrade to the latest release:

```bash
uv tool install --force jira-helper --index https://xohzzwn6kcj9.github.io/gh-dist/simple/
```

Runtime dependencies (typer, httpx, PyYAML) resolve from PyPI, so `pypi.org` must be
reachable over HTTPS; only `jira-helper` itself comes from this index.
