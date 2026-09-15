# Probe: exit-code-behavior (uv 0.12.14)

## Probe metadata

| Field                 | Value                                      |
|-----------------------|--------------------------------------------|
| Pattern               | `exit-code-behavior`                       |
| PM                    | uv                                         |
| PM version under test | 0.12.14                                    |
| PM version tested     | 0.12.14                                    |
| Schema version        | 1.2                                        |
| Generated at          | 2026-09-15T02:32:00Z                       |
| Categories            | tree_command, install_command              |

## What this probe tests

uv 0.12.14 changed exit-code semantics for `uv tree` and install
commands:

- **Exit code 1** — expected failures: resolution errors, missing
  packages, constraint conflicts that uv can report cleanly.
- **Exit code 2** — operational/internal failures: I/O errors,
  binary corruption, unexpected runtime errors.

The Mend Unified Agent invokes `uv tree` (and install commands) and
parses both the exit code and the diagnostic output to decide whether
resolution succeeded. Before 0.12.14, missing-package scenarios
returned exit code 2 (operational), causing Mend to treat them as
tool failures. After 0.12.14, they return exit code 1 (expected
failure), which Mend must interpret differently to avoid false
tool-failure escalations.

The diagnostic output format also changed in this release, which
affects Mend's log-scraping logic.

## Project structure

```
exit-code-behavior-20260915-023200/
├── pyproject.toml          PEP 621 manifest
├── uv.lock                 Lockfile (canonical uv format)
├── .python-version         Python 3.11 (Mend reads this first)
├── src/
│   └── exit_code_probe/
│       └── __init__.py
├── README.md               This file
└── expected-tree.json      Ground truth for downstream comparison
```

## Dependencies

### Main dependencies (happy-path, exit code 0)

| Package  | Version  | Transitives                          |
|----------|----------|--------------------------------------|
| httpx    | 0.27.2   | anyio, certifi, httpcore, idna,      |
|          |          | sniffio                              |
| click    | 8.1.7    | colorama (Windows only, marker)      |

Transitive chain:
- httpx -> anyio -> idna, sniffio
- httpx -> certifi
- httpx -> httpcore -> certifi, h11
- httpx -> idna
- httpx -> sniffio
- click -> colorama (sys_platform == 'win32')

### Optional extra: `[crypto]` (NOT activated in happy path)

| Package      | Version | Transitives   |
|--------------|---------|---------------|
| cryptography | 42.0.8  | cffi          |
| cffi         | 1.17.1  | pycparser     |

The `[crypto]` extra is defined but NOT activated in the main
dependency list. This exercises the exit-code-1 path: invoking
`uv tree --extra crypto` when cryptography is not installed would
previously return exit code 2 (Mend reads: tool error). With
0.12.14, it returns exit code 1 (Mend reads: expected failure,
dependency not installable in this context).

The expected-tree.json encodes the **happy-path** tree (no extras
activated) — this is what Mend should detect when it runs
`uv tree` without extra flags.

## Exit-code test scenarios

| Invocation                      | Pre-0.12.14 | Post-0.12.14 | Mend impact                      |
|---------------------------------|-------------|--------------|----------------------------------|
| `uv tree` (happy path)          | 0           | 0            | No change — success detected OK  |
| `uv tree --extra crypto`        | varies      | 1            | Mend must not treat 1 as fatal   |
| `uv pip install nonexistent`    | 2           | 1            | Mend must re-map exit-1 handling |
| Internal tool failure           | 1 or 2      | 2            | Mend must still treat 2 as fatal |

## What Mend must detect (happy path)

All packages from `uv.lock` that are part of the default resolution
(no extras activated):

- httpx 0.27.2 (direct)
- click 8.1.7 (direct)
- anyio 4.4.0 (transitive via httpx)
- certifi 2024.8.30 (transitive via httpx and httpcore)
- h11 0.14.0 (transitive via httpcore)
- httpcore 1.0.5 (transitive via httpx)
- idna 3.8 (transitive via httpx and anyio)
- sniffio 1.3.1 (transitive via httpx and anyio)
- colorama 0.4.6 (transitive via click, marker: sys_platform == 'win32')

## Mend failure modes to watch

- Mend treats exit code 1 from `uv tree` as a fatal operational
  failure and skips the entire scan (pre-0.12.14 behavior).
- Mend fails to parse the new diagnostic output format, losing all
  package names/versions.
- Optional-dependency packages (cryptography, cffi, pycparser) are
  erroneously included in the main tree when the `[crypto]` extra
  is not activated.
- colorama is included unconditionally (marker `sys_platform ==
  'win32'` dropped).

## Python version detection

Mend reads `.python-version` (single-line `3.11`) before
`pyproject.toml`'s `requires-python = ">=3.11"`. Both declare the
same version so there is no divergence. Mend's Python-version
detection will resolve to Python 3.11.

Reference: `plugins/mend-knowledge/skills/mend-sca/references/
python-version-detection.md` — PIP precedence chain.

## Mend config

**Bucket B — no `.whitesource` emitted.**

uv has partial dynamic Python version detection from `.python-version`
and `pyproject.toml`. This probe does not target a versioning
regression (it targets exit-code behavior), so the Bucket B default
applies: skip `.whitesource`. The uv tool itself is not in the
`install-tool` list and cannot be pinned via `versioning`.

If future scans show Python-version drift, add:

```json
{
  "scanSettings": {
    "versioning": { "python": "3.11" }
  }
}
```

## Resolver knowledge provenance

- Resolver file: `python.md`
- Upstream URL: https://raw.githubusercontent.com/whitesource/unified-agent/integration/.claude/knowledge/resolvers/python.md
- Fetched at: 2026-09-15T02:31:16+00:00
- Upstream SHA: 877d6d848391d838fb31b38dadf37d3ad0696cbc

Key UA behavior (from resolver file):
- uv projects are detected via `MEND_SCA_UV_PROJECTS` env var
  (semicolon-separated paths); manifests in those paths are removed
  from the pip-resolver scan list.
- The actual dependency graph comes through the pip-compatible path
  (`pip download` or `pipdeptree --json` when hierarchy is needed).
- `uv tree` output format changes in 0.12.14 may affect the UA's
  pre-step or diagnostic parsing — this probe targets that boundary.
