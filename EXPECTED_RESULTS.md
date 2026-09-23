# Expected SCA results — ground truth (Python / pip)

Lock-less `requirements.txt`, so generation resolves ranges/unpinned entries and
transitives.

## Summary

| bucket | packages |
|--------|----------|
| Vulnerable | `PyYAML@5.1`, `Jinja2@2.10`, `urllib3@1.24.3` (range resolved), `requests@2.20.0`, `idna@2.7` (transitive), `Werkzeug@0.15.3` (via `-r`) |
| Healthy | `six@1.17.x` (latest), `MarkupSafe@…` (transitive of Jinja2), `python-dateutil@2.8.x`, `PySocks@1.7.1`, `chardet@3.0.4`, `certifi@…`, `typing-extensions@4.4.0` |
| Unresolved | none |

`colorama` carries a `sys_platform == "win32"` marker and is skipped on Linux
(absent from results, or healthy at 0.4.6 if listed textually; either is fine).

## Vulnerabilities
- **`PyYAML@5.1`** — arbitrary code execution via `full_load`, `CVE-2020-1747`
  (and `CVE-2020-14343`), fixed in 5.4. Zero deps.
- **`Jinja2@2.10`** — sandbox escape `CVE-2019-10906` and ReDoS `CVE-2020-28493`.
  Pulls `MarkupSafe` (healthy transitive).
- **`urllib3`** — the range `>=1.24,<1.25` resolves to `1.24.3`, which is still
  vulnerable (`CVE-2020-26137` CRLF, `CVE-2021-33503` ReDoS; both fixed later).
  Zero deps. This is the key check that generation resolves a range **and** that
  a resolved range can itself be vulnerable.

## Healthy
- **`six`** — unpinned, resolves to the latest (1.17.x at the time of writing), no advisories. Tests
  the unpinned → resolved path.
- **`MarkupSafe`** — transitive of `Jinja2`, healthy. Proves transitive discovery.

## Pass / fail
- PASS: PyYAML, Jinja2, and urllib3 vulnerable; six and MarkupSafe healthy; 0
  unresolved.
- FINDINGS: urllib3 or six left **unresolved** (range/unpinned not resolved),
  MarkupSafe missing (no transitive discovery), 0 vulns (false all-clear), or any
  invented version. The exact urllib3 patch in the 1.24 range may be 1.24.2/1.24.3;
  "urllib3 in the 1.24.x range, vulnerable" is the pass condition.

## New edge case (regression re-test) — `~=` compatible-release pin
`requirements.txt` adds `python-dateutil~=2.8.0`.
- **PASS:** `python-dateutil@2.8.x` is healthy, and its transitive `six` resolves
  healthy. Tests the `~=` operator.

## Round 2 edge cases

### A. Extras + vulnerable transitive (`requests[socks]==2.20.0`)
- **`requests@2.20.0`** is vulnerable (`CVE-2023-32681` Proxy-Authorization
  leak, fixed 2.31.0; `CVE-2024-35195`, fixed 2.32.0; `CVE-2024-47081`, fixed
  2.32.4).
- Its pin `idna>=2.5,<2.8` forces **`idna@2.7`**, vulnerable (`CVE-2024-3651`
  ReDoS, fixed 3.7). This is the **vulnerable transitive** check.
- The `[socks]` extra adds **`PySocks@1.7.1`** (healthy). `chardet@3.0.4` and
  `certifi` (latest) are healthy transitives. `urllib3` stays in 1.24.x (its
  `<1.25` pin intersects the root range).
- **PASS:** requests and idna vulnerable, idna marked transitive; PySocks
  present and healthy.
- **FAIL:** `requests[socks]` left unresolved (extras not parsed), PySocks
  missing (extra ignored), or idna missing/healthy.

### B. `-r requirements-dev.txt` include
`requirements-dev.txt` holds `Werkzeug==0.15.3`, vulnerable (`CVE-2023-25577`
multipart DoS and `CVE-2023-23934`, fixed 2.2.3; `CVE-2024-34069`, fixed 3.0.3).
Zero deps.
- **PASS:** `Werkzeug@0.15.3` vulnerable. (If the scanner also treats
  `requirements-dev.txt` as its own manifest it may appear twice; that is
  acceptable but worth noting.)
- **FAIL:** Werkzeug absent (`-r` not followed).

### C. Option line (`--index-url https://pypi.org/simple`)
- **PASS:** ignored; no package named `--index-url`, generation unaffected.
- **FAIL:** parse error / whole generation fails, or an invented package.

### D. Environment marker (`colorama==0.4.6; sys_platform == "win32"`)
- **PASS:** absent (skipped on Linux) or healthy at 0.4.6.
- **FAIL:** an entry whose name contains the marker text, or unresolved.

### E. Name normalization (`typing_extensions==4.4.0`)
PyPI's canonical name is `typing-extensions`.
- **PASS:** healthy at 4.4.0 under either spelling.
- **FAIL:** unresolved because `typing_extensions` didn't match.

### Round 2 pass / fail (combined)
- PASS: 6 vulnerable (PyYAML, Jinja2, urllib3, requests, idna, Werkzeug);
  six, MarkupSafe, python-dateutil, PySocks, chardet, certifi,
  typing-extensions healthy; 0 unresolved.

## Round 3 edge cases — dev-only transitives (`pipenv/`)
A second, committed-lock project under `pipenv/`: `Pipfile` declares
`markdown==3.3` (packages) and `pytest==4.6.0` (dev-packages); `Pipfile.lock`
lists pytest's tree only under `develop`.
- **PASS:** `markdown@3.3` healthy **PROD** direct; `pytest@4.6.0` healthy
  **DEV** direct; `py@1.8.0` vulnerable (`CVE-2022-42969` ReDoS, no fix)
  **DEV** transitive; `pluggy`, `atomicwrites`, `attrs`, `more-itertools`,
  `packaging`, `pyparsing`, `wcwidth` healthy **DEV** transitives.
- **FAIL:** `py@1.8.0` (or any develop-only package) marked PROD.
