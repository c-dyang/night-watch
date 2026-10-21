# Provenance: the evidence strength behind each claim

> This file has exactly one purpose: **let the reader know which sentences are measured and which are my reasoning.**
> The most expensive flaw in a whitepaper is not a wrong conclusion — it is a reader who **cannot tell those two apart**.

---

## 1. Three evidence tiers

| Tier | Definition | Not this tier |
|---|---|---|
| **measured** | Ran in this implementation, artefacts inspectable, repeatable | "I remember it being like that" / "it usually works that way" |
| **observed** | A clear phenomenon seen once, no controlled experiment | Repeated sightings upgrade it to measured, but the trace must be kept |
| **claimed** | Design reasoning, not yet validated | Anything phrased as "should work" |

**Writing discipline**: unmarked text in the body is measured or observed by default; every claim listed as `claimed` below must carry its own warning in the corresponding body passage (the execution chapter carries a header warning already).

---

## 2. Invariant distribution by tier

| Domain | Entries | measured | observed | claimed |
|---|---|---|---|---|
| Memory (INV-M) | 11 | 7 | 1 | 3 |
| Execution (INV-E) | 6 | 0 | 0 | **6** |
| Device (INV-D) | 10 | 8 | 0 | 2 |
| Cross-cutting (INV-X) | 5 | 5 | 0 | 0 |
| **Total** | **32** | **20** | **1** | **11** |

> Note: `INV-D4` (zero-cost gate) is marked "measured + claimed" — zero cost is measured, the value of its graded criteria is still claimed. It counts as measured here and also appears once in the unvalidated list in §3.

**How to read it**: claims cluster in two places — the execution domain as a whole (all six unvalidated), and five "trust and tempo" entries: settling order (INV-M4), score-free blind lists (INV-M8), two clock speeds (INV-M10), waking instead of polling (INV-D5), and the number of denoising layers (INV-D9). Those two areas are the most interesting and the least validated parts of the architecture.

---

## 3. Unvalidated list (every `claimed` entry)

| ID | One line | Why not yet validated |
|---|---|---|
| INV-M4 | Settle before clear | The dependency is reasoning; a destructive "reverse the order" test would really lose data |
| INV-M8 | Blind-review lists carry no confidence marks | No A/B: does scoring distribution differ with prior scores shown vs hidden |
| INV-M10 | Two clock speeds | Needs long-run samples to be meaningful |
| INV-E1 | Judgement must come from an external independent LLM | No controlled comparison of in-context self-review vs external judging |
| INV-E2 | Audit input must be the raw snippet | No measurement of how verdicts differ between digest input and raw input |
| INV-E3 | Post-hoc correction outranks pre-routing | Rule explosion and suppressed exploration are not quantified |
| INV-E4 | Rating evidence comes only from production sessions | No comparison of ledger quality between "evidence-based" and "self-assessment" |
| INV-E5 | Two latency classes | No measurement of the synchronous verdict's effect on main-path latency |
| INV-E6 | Pre-checks must not become approvals | No measurement of action pass-rate change with pre-checks enabled |
| INV-D5 | Wake, do not poll | No cost comparison of polling vs waking |
| INV-D9 | Three denoising layers, never four | Observing "layers masking each other" requires accumulated false positives |
| INV-D4 (partly) | The value of the gate's **graded criteria** | Zero cost is measured; judgement quality of "worth acting on" has no control yet |

---

## 4. Three experiments that would promote claims to measurements

To move this document from `v0.1-draft` to `v1.0`, these three need to be run in order. Readers can run them too.

### Experiment 1: verdict consistency (validates INV-E1 / E2)

- **Method**: take 20 real snippets, judge three ways: (a) self-review in the same context, (b) external + raw snippet, (c) external + digest input.
- **Metrics**: number of problems found per arm; share confirmed by human review; `class` agreement across two runs of the same arm.
- **Expectation** (claimed): (a) finds the fewest; (c) finds clearly fewer than (b); repeat agreement (b) > (c).
- **Decision rule**: if (b) is not significantly better than (a), INV-E1 is falsified and **must be rewritten outright**, not reworded.

### Experiment 2: pre-check value and cost (validates INV-E5 / E6)

- **Method**: run the same task batch twice, pre-checks on and off; record overreach/omission counts and total wall time.
- **Metrics**: change in incident rate, change in average duration, number of actions actually blocked (should be ≈0).
- **Expectation**: incidents down, duration up but acceptable, blocks at zero.

### Experiment 3: gate zero-cost and asymmetric errors (validates INV-D4 / D5)

- **Method**: replay one day of signals (normal traffic plus one injected real failure); count the gate's three states and the number of model calls.
- **Metrics**: model calls on a normal day (should be 0), wake latency for the injected failure, false wakes.
- **Expectation**: calls 0; injected failure wakes within the window; false wakes > 0 but cheap.

---

## 5. What this draft was checked against

| Evidence kind | Used for | Tier |
|---|---|---|
| Four consecutive days of inspection logs | Correcting architectural facts (module duties, full rule set vs rules enabled that day, self-heal breadth) | measured |
| Implementation source | Contracts and fields, exit-code semantics, rebuild logic | measured |
| One real incident post-mortem | Reverse-engineering INV-X1 / INV-D2 / INV-D3 | measured |
| One "dry run that ran for real" mistake | Reverse-engineering Appendix B #6 (process discipline) | observed |
| One provider bot-protection block | Reverse-engineering Appendix B #15 (classify failures) | observed |
| Everything else | Design reasoning | claimed |

---

## 6. Sanitisation applied before publication (so reviewers can check)

Abstract substitutions made between the internal implementation and this document (**renaming, not altering facts**):

| Internal | Used here |
|---|---|
| Concrete service names and ports | "tool gateway" / "endpoint" |
| Concrete hosts and network topology | "single machine" / "node" |
| Concrete scheduler and launcher implementations | "scheduler" / "hidden-window launcher" |
| Concrete models and vendors | "free/local model" / "main model" / "external judge" |
| Concrete file paths | Named by role (inject layer / index layer / …) |
| Personal profile and household data | Never appears |

**How a reviewer can check**: any "measured" claim should be traceable to an artefact and a source location in the implementation. If it is not, this table is lying.
