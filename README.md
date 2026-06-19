# Probe: workspace-exclusive-groups

## Purpose

Exercises **workspace-exclusive dependency groups** introduced in uv
0.11.x (specifically confirmed in 0.11.22). A workspace-exclusive group
is a `[dependency-groups]` entry declared at the **virtual workspace
root** that is NOT propagated to member packages.

This probe verifies that Mend SCA detects the workspace-root groups
(`integration`, `audit`) as distinct groups and does not:

- Silently drop them.
- Merge/flatten them into member package dependencies.
- Incorrectly attribute them to a member.
- Fail to parse `[dependency-groups]` on a virtual root with no
  `[project]` section.

## Structure

```
workspace-exclusive-groups-<timestamp>/
├── pyproject.toml           # Virtual workspace root — no [project]
├── uv.lock                  # Shared lockfile (uv 0.11.22 format)
├── .python-version          # 3.11 (Mend PIP-chain precedence)
├── packages/
│   ├── core/
│   │   ├── pyproject.toml   # Member: depends on click
│   │   └── src/core/__init__.py
│   └── api/
│       ├── pyproject.toml   # Member: depends on flask + core (workspace)
│       └── src/api/__init__.py
├── README.md
└── expected-tree.json
```

## Workspace-exclusive groups

The root `pyproject.toml` defines two groups:

```toml
[dependency-groups]
integration = [
    "httpx>=0.27",
]
audit = [
    "pip-audit>=2.7",
]
```

Neither `packages/core/pyproject.toml` nor `packages/api/pyproject.toml`
declares these groups. They exist only at the workspace root.

## Key lockfile feature: `[package.dependency-groups]`

The root package entry in `uv.lock` uses the NEW key introduced in
uv 0.11.x:

```toml
[[package]]
name = "workspace-exclusive-groups"
version = "0.0.0"
source = { virtual = "." }

[package.dependency-groups]
audit = [
    { name = "pip-audit", specifier = ">=2.7" },
]
integration = [
    { name = "httpx", specifier = ">=0.27" },
]
```

This is intentionally NOT `[package.dev-dependencies]` (the legacy key).
A Mend resolver that only reads the legacy key will drop the workspace-
exclusive groups entirely.

## Packages

| Package | Version | Group | Source |
|---------|---------|-------|--------|
| core | 0.1.0 | main (workspace) | local |
| api | 0.1.0 | main (workspace) | local |
| click | 8.1.8 | main | registry |
| flask | 3.1.1 | main | registry |
| blinker | 1.8.2 | main | registry |
| itsdangerous | 2.2.0 | main | registry |
| jinja2 | 3.1.5 | main | registry |
| markupsafe | 3.0.2 | main | registry |
| werkzeug | 3.1.3 | main | registry |
| httpx | 0.27.2 | integration | registry |
| anyio | 4.4.0 | integration | registry |
| certifi | 2024.8.30 | integration | registry |
| httpcore | 1.0.7 | integration | registry |
| h11 | 0.14.0 | integration | registry |
| idna | 3.10 | integration | registry |
| sniffio | 1.3.1 | integration | registry |
| pip-audit | 2.7.3 | audit | registry |
| packaging | 24.2 | audit | registry |
| pip | 24.3.1 | audit | registry |
| pip-api | 0.0.34 | audit | registry |
| requests | 2.32.3 | audit | registry |
| charset-normalizer | 3.4.0 | audit | registry |
| urllib3 | 2.2.3 | audit | registry |
| resolvelib | 1.0.1 | audit | registry |
| rich | 13.9.4 | audit | registry |
| markdown-it-py | 3.0.0 | audit | registry |
| mdurl | 0.1.2 | audit | registry |
| pygments | 2.18.0 | audit | registry |
| platformdirs | 4.3.6 | audit | registry |

Note: certifi, idna, and requests appear as transitive deps of both
`httpx` (integration group) and `pip-audit` (audit group). In the
expected tree they are listed once with the group of their primary
resolver path.

## Python version detection

Mend follows the PIP precedence chain for Python version detection.
`.python-version` (present in this probe, value: `3.11`) has HIGHER
precedence than `[project] requires-python` in `pyproject.toml`.

- **File used by Mend:** `.python-version` = `3.11`
- **Backup manifest:** members each declare `requires-python = ">=3.11"`
- Both declare Python 3.11 — no conflict.

## Mend config

**Bucket B — no `.whitesource` emitted.**

`python-uv` has partial dynamic version detection. Mend reads the Python
version from `.python-version` (highest priority in the PIP chain). The
`uv` tool itself is NOT in the `install-tool` list, so the uv version
cannot be pinned via `scanSettings.versioning`. Dynamic detection covers
the Python version. No branch scoping, project-token routing, or
resolver-toggle behavior is under test here.

## Mend failure modes under test

1. **Workspace-exclusive groups silently dropped** — `[package.dependency-groups]`
   in the lockfile not recognized; httpx and pip-audit not reported.
2. **Legacy key confusion** — resolver reads `[package.dev-dependencies]`
   but not `[package.dependency-groups]`; groups disappear.
3. **Groups merged into member deps** — httpx or pip-audit attributed to
   `core` or `api` instead of the virtual root.
4. **Virtual root parse failure** — root has no `[project]` section;
   parser errors out and skips the entire workspace.

## Resolver note (Mend UA)

The Mend UA Python resolver (as of the fetched knowledge at
`2026-06-19T07:41:29Z`, SHA `877d6d84`) does not document explicit
handling of PEP 735 `[dependency-groups]` nor workspace-exclusive groups.
The resolver knowledge documents `### UV Project Filtering` via
`MEND_SCA_UV_PROJECTS` env var. Mend flows uv projects through the Pip
resolver path. Whether it reads `[package.dependency-groups]` from the
lockfile is therefore exploratory — this probe is designed to catch the
regression either way.

## Probe metadata

```json
{
  "pattern": "workspace-exclusive-groups",
  "pm": "uv",
  "pm_version_tested": "0.11.22",
  "generated_at": "2026-06-19T07:41:59Z",
  "resolver_sha": "877d6d848391d838fb31b38dadf37d3ad0696cbc",
  "resolver_fetched_at": "2026-06-19T07:41:29Z"
}
```
