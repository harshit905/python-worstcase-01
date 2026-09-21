# Expected SCA results — ground truth (Python / pip)

Lock-less `requirements.txt`, so generation resolves ranges/unpinned entries and
transitives.

## Summary

| bucket | packages |
|--------|----------|
| Vulnerable | `PyYAML@5.1`, `Jinja2@2.10`, `urllib3@1.24.3` (range resolved) |
| Healthy | `six@1.16.x`, `MarkupSafe@…` (transitive of Jinja2) |
| Unresolved | none |

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
- **`six`** — unpinned, resolves to the latest (1.16.x), no advisories. Tests
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
