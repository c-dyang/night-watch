# Evolution: from two requirements to three domains

> This appendix does one thing: **lay out, in time, how it actually grew** — each row is *real trigger → mechanism added → invariant it produced*.
> No theoretical phases, and no tidying the history for a cleaner story; the mechanisms interleaved, and the table interleaves them honestly.

---

## 1. Timeline (ordered by real triggers)

| Date | Trigger (real need / real incident) | Mechanism added | Invariant |
|---|---|---|---|
| 2026-06 | Context filled up with content; "a conclusion given is a search not run" | Memory layering: inject → index → distill → raw stream | `INV-M1` |
| 2026-07 | Reading the raw stream first meant wading through ~99% tool noise | Discipline: retrieval drills down **in order, no skipping** | `INV-M2` |
| 2026-07 | A derived index was hand-edited and contradicted the body | One source of truth + derived indexes **rebuilt wholesale** (no incremental) | `INV-M3` / `INV-M5` |
| 2026-07-16 | Recursive scanning to find a file was slow and unreliable | Local file location always goes through the index (no recursive walks) | `INV-M11` (adjacent discipline) |
| 2026-07-29 | Self-review kept defending its own prior choice | **External independent verdicts** shipped (separate process, prompt and ledger) | `INV-E1` |
| 2026-07-31 | "Checks I remembered to run" were missed and nobody noticed | Hooks **instrumented automatically** (voluntary → automatic) | `INV-D1` (liveness from artefacts) |
| 2026-08-03 | Day one of blind review produced a **wrong injection** (log/IME experience into a dev task), scored −1 by the external judge | Verdict input gained "current task domain"; async; redaction | `INV-E2` / `INV-E4` |
| 2026-08-04 | The experience store only ever grew; cold modules were squeezed out | Confidence ledger + tombstones + **"not adopted" never scores negative**; `[LOCK]` exemption | `INV-M7` / `INV-M8` |
| 2026-08-08 | The single-source judge model **returned 503 and sat silently for 24h** | Dual-source fallback + failure recorded | `INV-X5` |
| 2026-08-10 | Judgement frequency ran away | An automatic switch (calm-counter triggered, not event-triggered) | `INV-E5` |
| 2026-08-11 | Event-driven triggering pushed self-audit to every 20–30 min (24 archives in one day) | **Event-driven triggering vetoed**; only periodic + 24h backstop kept | `INV-E3` / `INV-E5` |
| 2026-08-11 | The audit could see text, not side effects | Architecture assessment: **freeze additions**; reposition to "stable operation + low-frequency spot checks" | `INV-E2` (known boundary) |
| 2026-08-13 | A session lived ~9h and the monitor cost more than it was worth | Monitor deleted (no illusion of "always online") | `INV-D5` (cost follows events) |
| 2026-08-14 | Host swap: memory and correction had to come along | Host adaptation layer (plugins) + reuse files without copying | `INV-X2` |
| 2026-09-02 | A session hand-off block carried "yesterday's conclusions"; the new session answered from the digest | Hand-off block reduced to **pure index** (signposts, not answers) | `INV-M1` |
| 2026-09-11 | A config rewrite **silently dropped** a third-party key from the credentials file | Single source of truth + explicit env fallback + a method for dating the loss | `INV-M3` |
| 2026-10-18 | Patrol depended on scheduled jobs; failures never entered the gate and nobody picked them up | Outcome signals → **zero-cost gate** → stepwise handling; rule set with on/off switches | `INV-D4` / `INV-D6` |
| 2026-10-21 | A broken test stub let a script **really stop the live service for ~3 minutes**, until the watchdog revived it | Executors report their own outcome; liveness from artefacts only; write authority separated from the supervised | `INV-X1` / `INV-D1` / `INV-D3` |
| 2026-10-21 | The question "why does any of this exist" got answered properly | Starting point collapsed to **two hard requirements** (unbounded task scope / single entry) | whole table |
| 2026-10-22 | Major host-version migration (resolution layer rebuilt, config carrier replaced, credential model changed) | Rollback manifest + post-migration verification; generation-compatible session streams | `INV-X2` / `INV-X3` |
| 2026-10-22 | Found that a single "strength" tier conflated **running** with **reasoned** | Evidence split into two dimensions (stance × degree of quantification `Q0–Q3`) | `06-invariants.md` §0 |

---

## 2. Two observations (not conclusions)

1. **The three guards were not built in phases; they interleaved.** Memory came first (June), execution next (Jul–Aug), device last (October) —
   but the triggers interleaved: "a derived index was hand-edited" and "a silent failure went unnoticed" happened within the same week.
   **Telling it by domain distorts it; telling it by date does not.**
2. **Almost every invariant's first evidence is an incident.** Of the 21 rows above, 15 have "something actually broke" in the trigger column.
   That is also why "evidence stops at `Q1`" is not embarrassing: **incidents supply the evidence; quantification is the debt the designer owes.**

---

## 3. One host migration, summarised

| Item | Before | After | Still effective? |
|---|---|---|---|
| Dependency resolution | One flat directory | Per-package links + rollback manifest | **Yes** after rebuild (reversible with the manifest) |
| Configuration carrier | A single config file | One-shot import → layered patches | **Yes, but semantics changed** (the original file is no longer read after import) |
| Credential supply | Process read the store | Isolated home mounts no stored auth; must be injected explicitly | **Broke once** (the voice chain went down), fine after injection |
| Session stream format | Old generation | New generation (fields extended) | **Yes** (read by generation-agnostic glob) |
| Memory and invariants | Files + conventions | Files + conventions | **Yes, unchanged** |

**The conclusion of this table**: what actually survives a host swap is not code — it is **file formats and invariants**, exactly as `INV-X2` predicted.

---

## 4. How to use this appendix

- To gauge an invariant's **credibility**: look at its trigger row in §1 — an incident, or a reasoning step?
- To reproduce the architecture: **do not replay the timeline**; follow the dependency graph in `06-invariants.md` §5.
- To assess migration risk: read §3, especially the row that says "semantics changed while the file still exists" — that is where verification is easiest to fool.
