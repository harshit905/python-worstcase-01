# SCA test repo — Python (pip), lock-less, worst case

Lock-less (`requirements.txt` with ranges and an unpinned entry, no pinned lock),
so the scanner must resolve versions and transitives. Worst-case elements: a
range that resolves to a still-vulnerable version, an unpinned entry, and a
transitive dependency.

See `EXPECTED_RESULTS.md` for the ground truth.

## Run it
1. New GitHub repo, e.g. `harshit905/sca-test-python`.
2. `git remote add origin <url>` then `git push -u origin main`.
3. Scan in CodeAnt, compare to `EXPECTED_RESULTS.md`.

Do NOT add a fully-pinned lock.
