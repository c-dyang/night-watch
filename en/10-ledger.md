# The Ledger (implemented) · 事务台账（已实现）

> The first nine volumes explain **why it is built this way**; this one is the first **module that actually runs and carries machine assertions**.
> It is not a new architecture — it is a slice of the previous invariants applied to one short segment: *event → timed reminder → controlled execution*.
>
> Stance: **broad detection, narrow execution, minimal traces.** Reasoning may be long; what lands on disk must be very short.

- Version：`v0.1-draft` ｜ Author：`c-dyang` ｜ Docs license CC BY 4.0 ｜ Code license MIT
- Size：19 scripts / 8 JSON contracts / **45 machine assertions** / 3 output channels (instant · briefing · calendar & to-do)

---

## 0. Why a separate volume

The evidence in the earlier volumes clusters at `Q1` (artefacts, no control), because most mechanisms cannot take a control: destroying a memory chain or building a bad classifier has a real cost.

The ledger is different. Its core is a handful of **pure functions** (normalise, fingerprint, decide, derive), so for the first time there is something that can actually be ablated:

| What this volume can offer | What the earlier volumes could not |
|---|---|
| Remove one rule from code → the assertion goes red, reproducibly | Review incidents after the fact |
| Same input, two implementations → a fixed numeric difference | Describe symptoms |
| Each of the 45 assertions points at a "must not", not a "should" | Discipline lives in prose, held by willpower |

So the writing order differs: **assertions first, rationale second.** Every claim below hangs on a script and a fixture, not on a citation.

---

## 1. Kernel: one sentence and three borders

> **Do not persist the model's unverified understanding; persist only traceable facts and experience. Do not limit task scope; limit only the boundary of behaviour and effect.**

Three borders, all prohibitions:

| # | Prohibited | Why |
|---|---|---|
| ① | Persisting "the model's understanding" | Understanding drifts, and **drifts silently** ([INV-X5]) |
| ② | Putting judgement and execution on the same side | Self-audit always defends its own choice ([INV-E1]) |
| ③ | Giving the loose side a hand | A loose table may be loose because it can only notify (I9 below) |

Border ③ is the genuinely new one in this volume; its origin is in §5.

---

## 2. The chain: the life of one event

```
External sources (four kinds)        ← every one is a recognisable event source
   │  copy hotkey / phone screenshot (post-OCR) / share sheet / voice transcript
   ▼
Inbox (single convergence point)
   │  deduplicated by filename watermark
   ▼
Extraction (pure)                    ← strip envelope, normalise, type, time anchor
   ▼
Fingerprint tkey (field fingerprint; excludes time bucket and reversibility)
   ▼
Gate (pure: five questions → 2×2 → level, and level ≤ cap)
   │
   ├─ drop ──────────────► take-off row only; no interruption
   ├─ push ──────────────► timed notification (see §4)
   │                        └─ has anchor → ticket; no anchor → never notify
   └─ execute ───────────► landing sheet (admission + acceptance) → guarded executor → record
                             └─ execution rights live only here, and only after a human signature
```

Source: [`../figures/fig6-ledger-loop.mmd`](../figures/fig6-ledger-loop.mmd)

The leverage is in the **funnel**, not in the entrances: entrances can be added without limit (I8), while the funnel's width is set by `cap` — and `cap` is decided by exactly one thing: **has a human signed for it**.

---

## 3. Three entrances and hard caps

The usual failure of such systems is entrances without categories: "I found this" eventually becomes "I asked you to do this". Here the entrance is fixed to three kinds, each with a **hard-coded** ceiling:

| Entrance | Meaning | Ceiling | May it execute? |
|---|---|---|---|
| `request` | You asked me to do it | `execute` | Yes — still needs a signed rule (§5 I1) |
| `record` | You asked me to note it | `push` | **Never**; execution requires a **new `request` event** |
| `found` | I noticed it | `push` | Same as above |

> **Invariant I2**: `level ≤ cap`, where `cap = min(entrance ceiling, rule-library ceiling)`.
> There is exactly one way to break it: letting the model fill in `level`. So `level` is computed by a pure function whose input contains no model opinion.

**Why `record` can never be upgraded in place**: "noted" and "decided to do" are different intentions. Allowing in-place upgrade lets the system quietly turn a note into a commitment. That costs one integer to express, and it is the cheapest gate in the whole design.

---

## 4. Five questions, 2×2, and three channels

### 4.1 Five questions

The gate does not ask "is this important" (that cannot be decided, [INV-E3]); it asks five answerable questions:

| Question | Reads | If missing |
|---|---|---|
| Which class? | Extracted type | Undecidable ⇒ `misc`, no ticket |
| When? | Time anchor | No anchor ⇒ never notify |
| Reversible? | Extracted flag | Irreversible ⇒ not in auto tier |
| Can it? | Action id + evidence tier | No action ⇒ notify only |
| Should it? | Red lines + budget | Out of bounds ⇒ `drop`, no follow-up (I7) |

**Undecidable is not guessed**: any unanswered question routes to the conservative branch, not to the "looks more like it" branch. There is a dedicated negative fixture (assertion: `derivable = true` with an empty landing sheet ⇒ red).

### 4.2 Three channels, nine time points

| Channel | Trigger | Ceiling | Shape |
|---|---|---|---|
| **Instant** | One of three take-off points is reached (prepare / anchor / back wall) | 1 per event, 6 per day | One line, ≤40 chars, must state *why now* |
| **Briefing** | The existing daily morning task | 3 lines | ≤25 chars per line |
| **Calendar / to-do** | Destination split: commitments with others vs own deadlines | — | Event with alarm / task with due date |

All three share the same brakes:

| Brake | Effect |
|---|---|
| Time-point dedup (ticket × point) | Never notify twice for the same moment |
| Decay ladder (raised once → shorter form → weekly → silence) | Stop nagging about the same thing |
| **Back wall**: due day always appears once, decay ignored | The only rule that overrides decay |
| Archive after 7 days overdue | A dignified exit for things nobody handles |
| **Push-list prohibition** | Ceilings are constants in code, not configuration |

> The prohibition looks minor but matters: once "just send me the whole list" is allowed, every ceiling above gets breached under the name of "just a bit more information".

### 4.3 A reminder is not a voluntary interruption

Arrival of an event **does not notify**; it only creates a ticket. Whether to notify is computed from time points, by a pure function:

```
candidate points = { due − lead time, explicit anchor, due (back wall) }
point reached and not yet fired → notify once
```

"Point reached" is a subtraction result, not a side effect of polling every N minutes. So **quiet periods genuinely cost nothing** ([INV-D5]).

---

## 5. The nine internal invariants (I1–I9)

These are internal to the module; each maps onto a book-level id. Two numbering systems coexist so that changing the module does not require renumbering the book.

| # | Invariant | Maps to | How it is broken |
|---|---|---|---|
| I1 | `binding = null` ⇒ `cap = push`, never `execute` | [INV-E1] | Hand-editing the cap |
| I2 | `level ≤ cap`, never over the ceiling | [INV-E6] | Letting the model fill `level` |
| I3 | `level = execute` ⇒ `derivable = true` | [INV-E7] | Execute first, derive later |
| I4 | Landing-sheet slot sources ∈ {raw, rule library, constant} | [INV-E4] | Adding a "model" source |
| I5 | Landing sheet written only by the generator (constant signature) | [INV-X1] | A script hand-writes one row |
| I6 | Take-off rows are append-only | [INV-X3] | Updating a row in place |
| I7 | Red line ⇒ `drop` and **do not ask** | [INV-E3] | Turning red lines into prompt text |
| I8 | Detection side is open; execution side changes need one human nod | [INV-E7] | Casually widening the cap |
| **I9** | **A loose table has no execution rights**: downgraded judgement may only exit to the notification layer | **candidate [INV-X8]** | Attaching an action to the loose side |

### Origin of I9 (the most valuable item in this volume)

To avoid losing capability during a migration, a **loose legacy path** was kept: entries the strict table cannot judge are handed to a form-filling table (extract fields → write calendar / attach reminder).

The obvious worry is double judgement. It does not happen because the order is fixed: strict first, loose only as fallback. But the deeper point is **why the loose table is allowed to be loose**:

> Its only exits are writing a calendar entry and attaching a reminder. Both are idempotent notifications — wrong ones can be deleted.
> Execution rights (actually doing something) exist only on the strict chain, and require a human signature.

So this is not "which judge is more reliable" but **decoupling judgement precision from effect authority**: the loose side may be loose because **it has no hands**. Its error costs one extra reminder; the strict side's error costs one real execution — an order of magnitude apart.

**This argument has a precondition that must be stated**: the loose side's safety comes not from the strict side's guard, but from its own action boundary. **The moment its exit becomes "can do a thing", it must use the same strict table.** So I9 does not say "loose is always safe"; it says "the loose exit must always be notification only".

---

## 6. Evidence: two real ablations + one batch upgrade

### 6.1 Ablation 1 — is the time bucket part of the fingerprint?

Should the same act at different times count as the same thing?

With the time bucket (morning/noon/evening) and reversibility included in the fingerprint:

| Configuration | Sample | Result |
|---|---|---|
| Bucket + reversibility **in** the fingerprint | 20 cases | One semantic class split into several tkeys; assertion red |
| Both **excluded** (plus media types ignore the object) | 20 cases | **Converges to 7 groups / 7 tkeys** |

**Conclusion**: a fingerprint carries "what + for whom", never "when" or "revocable". The latter two are **occurrence properties**, not **identity properties**. Putting them into identity means every occurrence becomes a new thing, and experience never accumulates.

This is a reproducible ablation: change two fields, run the same sample, output flips from red to green and group count converges to 7.

### 6.2 Ablation 2 — which normalisation for the self-hash?

The guard file needs self-verification (to prevent "casually widened"). Version one `pop`ped its own field before hashing; version two **zeroed** it. Same file:

| Normalisation | Digest | Matches the stored value? |
|---|---|---|
| `pop` the own field | `aa2f074c41dc826f` | No — **never self-consistent** |
| Zero it, then hash | `0328c3f6c9a42dde` | Yes |

The two spellings produce two different JSON strings, hence necessarily different digests — **a self-referential field must have exactly one specified normalisation, otherwise the check fails in the first second, and fails in a way that looks like a genuine corruption.**

The value of this class of bug: it is not a logic error, it is two reasonable spellings producing different bytes. Without a comparison you never find it.

### 6.3 Batch upgrade: several entries from `Q2` to `Q1`

Entries with machine assertions no longer rely on "I remember it behaves this way": the assertion must have gone red at least once. The following can move to `Q1` (artefacts + reproducible assertions) but **not to `Q0`**, because there is still no control measuring "how much worse without it" (see `07-sources.md` and change rule 3 in `06-invariants.md`):

| Entry | Reason for the upgrade |
|---|---|
| [INV-E6] pre-check must not degrade into approval | The decision function has 10 fixtures, including "ceiling must not be exceeded" negatives |
| [INV-E7] admission inputs are machine-produced | Landing sheet written by the generator with a constant signature; a hand-written row turns the assertion red |
| [INV-D5] wake, don't poll | Notifications fire from a subtraction; quiet periods make zero calls (reproducible) |
| [INV-D8] judging separated from delivery | Judging writes tickets; delivery reads tickets; file-level isolation |
| [INV-X3] sessions are cache, files are truth | All state in append-only logs; the process may stop and start at will |
| [INV-X5] silent failure is distinguishable from silent success | No artefact ⇒ explicit anomaly flag; `blocked` counted separately from "crashed" |

---

## 7. Four pitfalls found in real data

A continuation of `05-appendix-b-pitfalls.md` — all from the module's first day on real input.

| Pitfall | Symptom | Root cause | Criterion |
|---|---|---|---|
| **① Envelope treated as body** | 14 transcripts all typed identically and collapsed into one fingerprint | The file header is a markdown/YAML envelope (title + metadata + section names) and was fed in as content | **Strip the envelope before extraction**; also promote the sender/tags inside it into structured fields |
| **② Fingerprint computed but never persisted** | The weekly review always reported "no candidates", looking like "no patterns exist" | The fingerprint was only computed while debugging and never written to events | Criterion: **any derived field must be persisted with its subject**, or downstream stays empty forever |
| **③ Self-referential hash never consistent** | Guard verification failed on day one, looking exactly like "the file really was modified" | See §6.2 | A self-referential field needs one specified normalisation |
| **④ Same-day burst mistaken for a habit** | An item seen 3 times in one day was promoted to "pattern", with a suggested rule | Frequency counted without span | Criterion: **a habit must span ≥3 days**; same-day repetition is a burst |

All four share a property: **none is a logic error; each is a semantic mismatch that looks correct.** That is exactly where machine assertions pay off — not against typos, but by **turning semantic errors into reproducible red**.

---

## 8. Known limitations (no beautification)

| # | Limitation | Consequence | Plan |
|---|---|---|---|
| 1 | **Type detection is keyword-based** | New classes fall to `misc` ⇒ never notified | Only the weekly review can promote frequent `misc` into rules; slow |
| 2 | **Lead times are constants** (commitment 24 h / delivery 48 h / errand 72 h) | Things needing a week of preparation get one day | Derive from the historical median of "reminder → actually handled" |
| 3 | **Tickets do not retire themselves** | Unhandled tickets exit by decay + 7-day archive, i.e. by forgetting, not by judgement | Acceptable, but the library's cleanliness depends on how often a human looks |
| 4 | **The judge is still thin** | On a real batch everything was judged `push`; `drop` never fired | Missing a "undecidable ⇒ not worth notifying" exit (pending) |
| 5 | **Only the execution *refusal* branch was verified** | Both real calls returned `blocked` (missing source, missing network); `done` is uncovered | Wait for the first genuinely executable task |

Point 5 needs emphasis: **being able to refuse is not the same as being able to work.** Both real calls returned `blocked` with correct reasons (one identified a non-existent target disk, one identified that the handoff supplied no source). That proves the guard and the contract work — but **"successfully completed one execution" is not something this volume can claim to have verified.**

---

## 9. Minimal reproduction set

No host integration, no model required — only Python and a filesystem:

| Item | Count | Role |
|---|---|---|
| JSON contracts | 8 | event / take-off / landing / record / rule library / guard / templates / gate contract |
| Pure functions | 4 | normalise · fingerprint · decide · derive |
| Side-effecting | 3 | ingest · ticket creation · delivery |
| Machine assertions | **45** | extraction/fingerprint 20 · decision 10 · reminders 7 · guard 8 |

Run (sanitised):

```bash
python scripts/collect.py --inbox          # ingest (filename watermark dedup)
python scripts/extract.py                  # extract (pure; re-runs do not rewrite)
python scripts/intake.py                   # decide → take-off rows → tickets only if worthy
python scripts/due_scan.py --wave W2 --md  # briefing section
python scripts/w1_tick.py --dry-run        # timed reminder (compute only by default)
python tests/run_fixtures.py               # 45 assertions
```

**Everything defaults to dry-run**: executing scripts require an explicit `--exec`. That is not caution, it is how I8 is implemented — the default value is itself a gate.

---

## 10. Three sentences this volume wants to leave behind

1. **Write prohibitions as assertions, not as paragraphs.** Paragraphs are forgotten by the next change; assertions are not.
2. **The loose side must not have hands.** Loose and strict can coexist, provided their exits are not the same thing.
3. **Keep traces extremely short.** Reasoning may be long, but what lands on disk must be only "what happened + on what grounds", otherwise the library grows into a self-essay.

> Aligning with the book's stance: sessions can be wiped, because memory is not in the session; models can be cheap, because the hard parts moved into structure;
> **and "did this actually happen, and on what grounds may it happen" is governed by a ledger of its own.**
