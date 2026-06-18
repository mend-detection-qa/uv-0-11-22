# Probe: uv 0.11.22 Release Features

## Probe metadata

| Field              | Value                                          |
|--------------------|------------------------------------------------|
| PM                 | uv                                             |
| PM version         | 0.11.22                                        |
| Patterns exercised | marker-ordering-semantics, workspace-exclusive-groups, pylock-lock-version |
| Generated at       | 2026-06-18T23:31:54Z                           |
| Schema version     | 1.2                                            |
| Categories         | tree_structure, version_constraints, lockfile_format |

## What this probe exercises

This probe covers three uv 0.11.22 release changes in a single
workspace project.

### 1. marker-ordering-semantics

PEP 508 environment markers with compound conditions were generated
in a different ordering by the uv 0.11.22 resolver. This affects
how Mend parses the marker string from the lockfile.

Two compound-marker deps are declared at the workspace root in
`[tool.uv.dev-dependencies]`:

- `colorama>=0.4.6; sys_platform == 'win32' and python_version >= '3.11'`
  — Windows-only, compound `and` marker combining `sys_platform`
  and `python_version`.
- `uvloop>=0.19; (sys_platform == 'linux' or sys_platform == 'darwin') and python_full_version >= '3.11.0'`
  — non-Windows, compound marker combining `or` with
  `python_full_version`.

The `uv.lock` `resolution-markers` header reflects the three
resolution branches uv computed.

**Mend failure mode:** Mend may strip the compound marker to the
first condition only, or drop the package entirely on non-matching
hosts. The expected tree encodes the marker exactly as it appears
in the lockfile.

### 2. workspace-exclusive-groups

The workspace root is a **virtual root** (no `[project]` section).
It declares `[dependency-groups]` at the root level:

- `integration = ["httpx>=0.27"]`
- `audit = ["pip-audit>=2.7"]`

These groups are **NOT** present in any workspace member's
`pyproject.toml`. uv 0.11.22 changed how such workspace-exclusive
groups are discovered when building the dependency tree.

Two workspace members:
- `packages/core` — `click>=8.1`, no dependency groups.
- `packages/api` — `flask>=3.0` + `probe-core` (workspace ref).

**Mend failure mode:** Mend may not discover workspace-root
`[dependency-groups]` when the root is virtual, causing `httpx`
and `pip-audit` (and their transitives) to be silently dropped.

### 3. pylock-lock-version

`pylock.toml` (PEP 751 format) is present alongside `uv.lock`.
The `lock-version = "1.0"` field exercises the validation path
changed in uv 0.11.22. The `pylock.toml` covers the basic
`requests` + its transitives subset.

**Mend failure mode:** Mend's parser may not recognize `pylock.toml`
at all (falling back to `uv.lock` only), or may fail to parse the
`lock-version` field correctly. The expected tree is built from
`uv.lock` (the authoritative source); `pylock.toml` is the probe
artifact for the format-validation test.

## File layout

```
uv-0.11.22-release-features-20260618-233154/
├── pyproject.toml         workspace root (virtual, no [project])
├── uv.lock                shared workspace lockfile
├── pylock.toml            PEP 751 companion lockfile (lock-version probe)
├── .python-version        Python 3.11 (higher PIP-chain precedence than requires-python)
├── packages/
│   ├── core/
│   │   ├── pyproject.toml
│   │   └── src/core/__init__.py
│   └── api/
│       ├── pyproject.toml
│       └── src/api/__init__.py
├── README.md              this file
└── expected-tree.json     expected dependency tree
```

## Python version detection

`.python-version` declares `3.11` and takes higher precedence in
Mend's PIP detection chain than `pyproject.toml`'s
`requires-python`. Both files are consistent (>=3.11 / 3.11).
Mend will use `.python-version` to determine the Python runtime.

## Mend config

**Bucket B — no `.whitesource` required.**

uv has partial dynamic version detection from the manifest (via
`requires-python` in `pyproject.toml` and `.python-version`). The
uv tool itself is NOT in the `install-tool` list, so it cannot be
pinned via `scanSettings.versioning`. Only `python` is pinnable,
and dynamic detection covers this probe's requirements.

No `.whitesource` is emitted. No `whitesource.config` is needed
(the default `python.resolveDependencies=true` path is sufficient).

## Resolver knowledge note

The Mend UA Python resolver flows uv projects through the Pip
resolver path (see `### UV Project Filtering` in the resolver
knowledge). The `MEND_SCA_UV_PROJECTS` environment variable can
be used to exclude UV-managed project manifests from scan, but
this probe does NOT set it — we want all manifests scanned.

The resolver does NOT natively understand `[dependency-groups]`
(PEP 735) — this is a known limitation. The expected tree for the
workspace-exclusive groups pattern encodes what Mend WILL emit
(groups may be dropped or flattened), not the ground truth. The
probe README explicitly flags this so the downstream comparator
treats it as a known divergence.
