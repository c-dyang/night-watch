# Appendix B: Engineering Pitfalls

> The body text says "how it should be"; this appendix says "how we stepped on it".
> Each entry = **symptom / root cause / test**. The test column is for readers: when you see the symptom, use it to confirm whether it is the same pit.

- Inclusion rule: **transferable** (you will hit it after swapping language or host). Host-specific APIs and local environment trivia are out.
- Version: `v0.1-draft`

---

## B.1 Observation (failures you cannot see)

| # | Symptom | Root cause | Test |
|---|---|---|---|
| 1 | A task clearly failed, the scheduler reports success | The hidden-window launcher returns **its own** exit code (always 0) to the scheduler; the launched process's exit code is discarded | Make the executor `exit 1` and watch the scheduler's last result; if unchanged, you are hit |
| 2 | Artefacts are all there but the task is judged failed, or the reverse | The completion criterion reads **the model's text output** ("done") instead of the artefact | Silence the model while the artefact still appears; it should pass |
| 3 | The checker's own state becomes the health verdict | Observation point mid-chain; the dashboard is written by a middle stage | Kill the second half of the chain; if the dashboard stays green, you are hit |
| 4 | "Nothing has happened" starting some day | Auxiliary sessions are lazily created ⇒ no session is misread as no inspection | Compare session count with the day's artefact count: artefact present, session absent = normal |
| 5 | The archive clearly has content but search never finds it | Full-text search run directly over a compressed archive (or the index was never built) | Decompress and grep: hits, while search returns nothing ⇒ index missing |

---

## B.2 Execution (breaking things while acting)

| # | Symptom | Root cause | Test |
|---|---|---|---|
| 6 | "We only did a dry run", yet production was touched | The script invokes the side-effecting entry point through **variable indirection**; the stub replaced only the literal command | Before a dry run, replace the whole side-effect entry with "print only", and have a human check that line; grep-based stubbing is not stubbing |
| 7 | Running a self-heal twice breaks the environment | A non-idempotent action entered the whitelist (config edits, deletions, notifications) | Run it twice and compare outcomes; any difference means it is not idempotent |
| 8 | Edited code does not take effect after hot reload | Bytecode cache timestamps have whole-second resolution, so edits inside the same second look "unchanged" | Edit, trigger reload immediately, verify behaviour; clear the cache directory and diff the behaviour |
| 9 | Batch scripts produce mojibake and truncated commands | The file is written in one encoding (e.g. GBK) while the runtime code page is another (UTF-8) | Run the script standalone and look at its output; changing code pages changes the symptom |
| 10 | A parameter carries a leading space and a comparison never matches | Whitespace left over from variable interpolation (`%VAR% `); the comparison silently fails | Print both values quoted with their lengths: a length difference of 1 is the tell |

---

## B.3 Data (quietly turning dirty)

| # | Symptom | Root cause | Test |
|---|---|---|---|
| 11 | Deleted entries still show up in search | The index was updated **incrementally** and deletions were never handled | Delete one entry → rebuild wholesale → search again; it disappears after rebuild means you were hit |
| 12 | A stale file passes as a fresh artefact | The completion criterion checks "file exists" but not "updated this round" | Compare artefact mtime with the round's start time; earlier than start = hit |
| 13 | A whitelist silently stops matching after a dependency upgrade | Dependency not version-pinned; semantics changed so the filter no longer matches | Print and pin dependency versions; behaviour returning after a rollback confirms it |
| 14 | Parsing yields nothing, appearing as "no data" | Parsing relied on localised UI column headers ("Name", "Status"); a language change breaks the match | Locate fields by index/ID instead; still empty after a language switch means you are hit |

---

## B.4 External services (other people's boundaries)

| # | Symptom | Root cause | Test |
|---|---|---|---|
| 15 | A search source fails entirely and retrying does not help | The provider's bot protection (browser signature / header fingerprint) tripped — not quota exhaustion, and explicitly non-retryable | Read the error class and message; "do not retry" errors get worse with retries |
| 16 | Several sources in one batch show near-identical content | Press-release / content-farm contamination; multiple sources are not independent evidence | Within-batch dedup and domain-crowding check; a count collapse after dedup confirms it |

---

## B.5 Rules derived from these pits (mapped to the body)

| Pit | Invariant it produces |
|---|---|
| #1 | Executors must self-report their outcome [INV-D3] |
| #2 | Health verdicts come from the end-of-chain artefact, not model text [INV-D2] |
| #3 | Health verdicts come from the end-of-chain artefact [INV-D2] |
| #4 | An absent auxiliary session is not evidence of failure [INV-D10] |
| #5, #11 | Derived indexes rebuild wholesale and are disposable [INV-M3/M5] |
| #6 | A dry run must never reach a real side-effect entry point (process discipline) |
| #7 | Only idempotent actions self-heal, and must self-verify [INV-D6] |
| #8, #10 | Hot reload and string comparison must be verifiable (process discipline) |
| #12 | Freshness checks need a time baseline [INV-D2] |
| #13 | Dependencies must be pinned, or "whitelist" is an illusion (process discipline) |
| #14 | Parsing must not depend on localised presentation (process discipline) |
| #15 | External failures must be classified, never retried blindly [INV-E5] |
| #16 | Multiple sources are not multiple proofs; compare within a batch (process discipline) |

> Note: the entries marked "process discipline" were deliberately **not** promoted to invariants, because they constrain the person writing the code, not the structure of the system. That distinction is exactly why they live in the appendix.
