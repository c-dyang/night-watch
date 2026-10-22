# Provenance: how far the evidence behind each claim actually goes

> This file has exactly one job: **let the reader tell apart what was run, what was merely reasoned, and how far the run was quantified.**
> The most expensive failure for a whitepaper is not a wrong conclusion — it is that **the reader cannot tell these apart.**

---

## 1. Two dimensions

**Why a single "strength" tier was abandoned**: it forces a binary choice between "measured" and "claimed", while reality sits in between —
a mechanism may have **run continuously for months with complete ledgers**, yet **never once have been compared against "what if it were absent"**.
Calling that "measured" is overclaiming; calling it "claimed" erases it. So the tiers are split.

### Dimension 1 · Stance

| Value | Definition |
|---|---|
| **Running** | Historical artefacts exist, and the last 7 days still produced new ones |
| **Partial** | Artefacts exist but coverage is incomplete (manual steps / only some entry points) |
| **Convention only** | Discipline lives only in documents or prompts — no enforcement, no artefacts |
| **Not implemented** | Design only |

### Dimension 2 · Evidence (ordered by degree of quantification)

| Value | Definition | Counter-example (does not qualify) |
|---|---|---|
| **Q0 controlled** | Control group / ablation / comparable baseline, i.e. a numeric delta | "I switched it and it felt better" |
| **Q1 traceable, not quantified** | Reproducible artefacts and ledgers, **but no control** | "It has always been running", with no records |
| **Q2 observed, not quantified** | Concrete events support it, no systematic record | "that's probably how it behaves" |
| **Q3 reasoned only** | No runtime evidence, design argument only | any "usually it is like this" statement |

**Writing discipline**: the body defaults to `Q1` or above; any `Q3` entry must carry an explicit warning in the corresponding paragraph (this draft does so at the head of the execution volume).

> **Core stance** (stated here because it is the premise of the whole document):
> This architecture was **built up from real operation** — every invariant traces back to something that actually ran or actually broke, not to speculation.
> What is missing is **quantification** ("how much worse would it be without this"), not **evidence** ("did it ever run at all").
> Conflating the two makes readers misread "we never measured the delta" as "we never built it".

---

## 2. Distribution across both dimensions

### 2.1 By stance

| Domain | Entries | Running | Partial | Convention only | Not implemented |
|---|---|---|---|---|---|
| Memory (INV-M) | 11 | 8 | 1 | 2 | 0 |
| Execution (INV-E) | 6 | 4 | 2 | 0 | 0 |
| Device (INV-D) | 10 | 7 | 3 | 0 | 0 |
| Cross-cutting (INV-X) | 5 | 5 | 0 | 0 | 0 |
| **Total** | **32** | **24** | **6** | **2** | **0** |

### 2.2 By evidence

| Domain | Entries | Q0 controlled | Q1 traceable | Q2 observed | Q3 reasoned |
|---|---|---|---|---|---|
| Memory (INV-M) | 11 | 0 | 9 | 2 | 0 |
| Execution (INV-E) | 6 | 0 | 4 | 2 | 0 |
| Device (INV-D) | 10 | 0 | 8 (incl. D4) | 2 | 0 (D4's graded value is Q3) |
| Cross-cutting (INV-X) | 5 | 0 | 5 | 0 | 0 |
| **Total** | **32** | **0** | **26** | **6** | **0** |

> Note: `INV-D4` is layered — "the gate costs nothing" is Q1; "the graded criteria are worth acting on" is Q3.

**How to read this**: **not a single entry comes from a controlled experiment (Q0 = 0)**.
This is not an oversight; it is the evidential reality of this class of system — single machine, single user, where **you cannot deliberately break things to run an experiment** (you would really lose data or really stop a service).
What was actually achieved here is **26 entries at Q1** — reproducible artefacts, traceable and recomputable — plus 6 at Q2 (events exist, records do not).
"What to do about Q0 = 0" is answered in §5.

---

## 3. What is still missing (exactly two things)

| ID | In one line | What kind of gap | Can it be closed? |
|---|---|---|---|
| `INV-D4` (graded value) | Whether the gate's "is this worth waking anyone" judgement is any good | **Missing quantification** (needs a long tail of false alarms to sample from) | A function of time, not of design |
| **Behavioural-layer audit (re-check after sensitive writes)** | The audit can only see text, not side effects (registry-type writes) | **Not implemented** — the only named capability gap | Implementable, not yet scheduled |

**Three entries that used to be listed here, and why they left** (they are not gaps — the verdict is in):

| ID | Why it moved out |
|---|---|
| `INV-E1` externally independent verdicts | Running continuously (673 ledger entries); the earlier implementation measured **6/6 single-round correction loops**. Its **known boundary** is same-source blindness (§4) — that is not "unvalidated", it is "we know where the ceiling is" |
| `INV-E2` audit input is raw snippets | Same as above; the "feed it a digest instead" A/B was never run, but the boundary ("the audit sees text only") has been identified and accepted |
| `INV-E3` post-hoc correction over pre-routing | **Three empirical cases** (pricing / release / full-text search — all three errors surfaced only *after* execution) plus four mechanism-level failure modes. The only possible "control" would require first building a deliberately bad classifier — **which should not be done** |

**Six additional entries at Q2 are not gaps either — they are thin records** (`INV-M2 / M10 / E5 / E6 / D5 / D9`):
the mechanism runs and specific events can be pointed at, but nothing systematically logs them ⇒ the fix is **adding instrumentation**, not running a control experiment.

---

## 4. Known boundaries: we know where this stops

This is the section most worth a reviewer's attention: **not "no problems yet", but "the problems have been located".**

### 4.1 Three weaknesses (still in force)

1. **Same-source auditor** — the auditor and the audited are both LLMs; **semantic-level misremembering cannot be caught**. Independence buys freedom from "defending one's own prior choice", not freedom from a shared blind spot.
2. **The audit reads text, not behaviour** — it judges what was said, not what the system did (registry / filesystem side effects are invisible).
3. **The governance layer is itself a failure source** — audit chain, injection chain and gate chain can all break (a meta-problem).

### 4.2 The resulting repositioning

> **Freeze the architecture; stop adding to it.** Reposition from "error prevention" to "**stable operation plus low-frequency spot checks**".
> The only capability worth adding is the **behavioural-layer audit** (re-check after sensitive writes), see §3.

This was a deliberate contraction, not an admission of weakness: adding more mechanisms only makes weakness #3 worse.

---

## 5. Why there is not a single Q0, and what to do instead

A controlled experiment requires running the system **without** the mechanism, and two classes of entries here **forbid exactly that**:

| Class | Why a control is impossible |
|---|---|
| Destructive-ordering entries (e.g. `INV-M4` settle-before-clear) | The reverse-order experiment means **actually losing data** |
| Safety-boundary entries (e.g. `INV-E3` pre-routing) | The control means **deliberately building a bad classifier first** |

So the strategy is not "manufacture experiments", but: **state how far each entry's evidence reaches**, and hand the **feasible** experiments (§6) to anyone willing to run them — a reader can run a control in their own environment far more safely than this machine can.

---

## 6. Three experiments worth running (pushing Q2/Q3 toward Q0 or Q1)

### Experiment 1: verdict consistency (quantifying `INV-E1` / `E2`)

- **Method**: take 20 real snippets; judge each three ways — ① same-context self-audit ② external + raw snippet ③ external + digest input.
- **Metrics**: findings per group and the share confirmed by human review; classification agreement across two repeated runs of the same group.
- **Expectation** (Q3): ① finds the fewest; ③ finds clearly fewer than ②; repeat agreement ② > ③.
- **Decision rule**: if ② is not significantly different from ①, `INV-E1` is falsified and **must be rewritten as a whole**.

### Experiment 2: gate value and asymmetric errors (quantifying `INV-D4`)

- **Method**: replay one day of signal traffic (normal traffic + one injected genuine failure); record the gate's three-state distribution and model call count.
- **Metrics**: model calls on a normal day (expect 0), wake-up latency for the injected failure, number of false wakes.
- **Expectation**: calls 0; injected failure wakes within the window; false wakes > 0 but cheap.

### Experiment 3: blind-review de-labelling A/B (quantifying `INV-M8`)

- **Method**: the same batch of entries, reviewed blind twice — once with prior confidence labels, once without; compare score distributions and how often new entries rise.
- **Metrics**: significance of distribution difference; share of new entries entering the pool within two weeks.
- **Expectation**: the labelled group reproduces high scores more often (halo effect).

---

## 7. What this draft was checked against

| Source type | Used for | Strength |
|---|---|---|
| Four consecutive days of patrol day logs | Correcting architectural facts (module duties, rule set and what was enabled that day, width of the self-heal surface) | Q1 |
| Implementation source code | Contracts and fields, exit-code semantics, rebuild logic | Q1 |
| One real incident post-mortem | Reverse-engineering `INV-X1` / `INV-D2` / `INV-D3` | Q1 |
| One "dry run that became a real run" mistake | Reverse-engineering appendix B #6 (engineering discipline) | Q2 |
| One search-provider block | Reverse-engineering appendix B #15 (failures must be classified) | Q2 |
| Historical assessment of the audit mechanism (three weaknesses) | §4 known boundaries | Q1 |
| Everything else | Design reasoning | Q3 |

---

## 8. Sanitisation applied before publication (so reviewers can check)

Abstractions made between this draft and the internal implementation (**substituting proper nouns, not editing facts**):

| Internal | Term used here |
|---|---|
| Concrete service names and ports | "tool gateway" / "endpoint" |
| Concrete hosts and network topology | "single machine" / "node" |
| Concrete scheduler and launcher implementations | "scheduler" / "hidden-window launcher" |
| Concrete models and vendors | "free/local model" / "main model" / "external verdict" |
| Concrete file paths | Named by role (inject layer / index layer / …) |
| Personal profile and household data | Never appears |

**How a reviewer can check**: pick any `Q1` claim; every one of them maps to an inspectable artefact and a source location in the implementation. If one does not, this table is wrong.
