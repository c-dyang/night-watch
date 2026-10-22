# Invariants

> This file is the skeleton of the whitepaper: every claim in the body text must point to one of the IDs below.
> **If a claim has no ID, it does not go into the body.**

- Version: `v0.1-draft` ｜ Author: `c-dyang` ｜ Docs license: CC BY 4.0
Two independent dimensions per entry (see the block below): **Stance** (is it running?) and **Evidence** (how far is it quantified?).

> This architecture was **built up from real operation**: every invariant traces back to something that actually ran or actually broke.
> What is missing is **quantification** ("how much worse without it"), not **evidence** ("whether it happened at all").
> Current spread: **Q0 = 0, Q1 = 26, Q2 = 6, Q3 = 0** — see `07-sources.md` §2; why Q0 is zero: §5.

**Citation rule**: the body always cites the ID form `[INV-M4]`; every invariant must be cited by at least one chapter — **an uncited entry is an orphan and gets deleted before release**.

---

## 0. Two dimensions (read this before the tables)

**Dimension 1 · Stance** — is the mechanism actually running?

| Value | Criterion |
|---|---|
| `Running` | Historical artefacts exist **and** the last 7 days still produced new ones |
| `Partial` | Artefacts exist but coverage is incomplete (manual steps / only some entry points) |
| `Convention only` | Discipline lives only in documents or prompts — no enforcement, no artefacts |
| `Not implemented` | Design only |

**Dimension 2 · Evidence** — running is not the same as quantified

| Value | Criterion |
|---|---|
| `Q0 controlled` | Control group / ablation / comparable baseline, i.e. a numeric delta |
| `Q1 traceable, not quantified` | Reproducible artefacts and ledgers, **but no control** |
| `Q2 observed, not quantified` | Concrete events exist, no systematic record |
| `Q3 reasoned only` | No runtime evidence |

---

## 1. Memory (INV-M)

| ID | Invariant | Reverse-engineered from | Symptom when violated | Stance | Evidence |
|---|---|---|---|---|---|
| INV-M1 | **Inject the map, not the content**: each round carries only indexes, digests and retrieval instructions | Injecting full text fills the context and suppresses retrieval ("a conclusion given is a search not run") | The new session believes it knows, while running on a stale digest | Running | Q1 |
| INV-M2 | **Retrieval drills down in order**: inject layer → index layer → distill layer → raw stream | Reading the raw stream first: ~99% tool noise, and the dirtiest layer ends up driving the judgement | Verdicts rest on single failed snippets and flip-flop | Convention only | Q2 |
| INV-M3 | **One source of truth plus derived artefacts**: derived indexes are never hand-edited, only rebuilt | Hand-edited index drifts away from the body | Search hits contradict the body, and nobody can tell which is stale | Running | Q1 |
| INV-M4 | **Settle before you clear**: persist first, rotate the session afterwards | Assuming clearing and settling are commutative | Unsettled content is genuinely lost, and **nothing reports an error** | Running | Q1 |
| INV-M5 | **Rebuild derived indexes wholesale, never incrementally** | Incremental updates strand entries after deletions and edits | The index drifts slowly and gets less accurate the longer it runs | Running | Q1 |
| INV-M6 | **Report the missing layer; do not silently re-run** | Auto-retry on failure hides the gap behind an "all filled in" appearance | One link is broken while every daily report is green | Running | Q1 |
| INV-M7 | **Experience entries must be refutable**: confidence ledger plus tombstone retirement, negative scores require a reason | An append-only experience store | Wrong entries live forever and read more authoritative as they age | Running | Q1 |
| INV-M8 | **Blind-review lists carry no confidence marks and no author self-assessment** | Scores anchor on prior scores and self-descriptions (halo effect) | Top entries stay on top forever; new entries never rank | Running | Q1 |
| INV-M9 | **Write and rate are separated**: the execution side only writes entries and evidence; the external side only changes confidence | One side both writes and rates | Self-assessment inflates; the ledger loses discriminating power | Running | Q1 |
| INV-M10 | **Two clock speeds**: fast signals (overreach / omission) corrected within 1 round; slow signals (confidence) applied weekly/monthly | Handling both signal classes at one tempo | Either correction arrives too late, or the ledger oscillates | Partial | Q2 |
| INV-M11 | **No full reads of a source of truth**: locate the section, read the section | Full reads continue after the file grows | One file eats the context budget | Convention only | Q1 |

---

## 2. Execution (INV-E)

| ID | Invariant | Reverse-engineered from | Symptom when violated | Stance | Evidence |
|---|---|---|---|---|---|
| INV-E1 | **Judgement must come from an external, independent LLM**: separate process, separate prompt, separate ledger | In-context self-review defends choices it already made | The audit always passes, which is the same as having none | Running | Q1 |
| INV-E2 | **Audit input is the raw snippet**, never a summary or retelling produced by the model under audit | Letting the audited party curate "what the auditor sees" | The auditor sees the polished version; overreach and omission are smoothed away | Running | Q1 |
| INV-E3 | **Post-hoc correction outranks pre-routing**: no upfront classification in open domains | Upfront classification is unenumerable ⇒ rule explosion, suppressed exploration, expired rules turn toxic | Rules multiply; every new situation gets blocked | Running | Q1 |
| INV-E4 | **Rating evidence comes only from production sessions**: no model self-reports, no document retellings | Substituting "is it well written" for "was it actually done" | The ledger rewards good writers, not correct behaviour | Running | Q1 |
| INV-E5 | **Two latency classes**: immediate correction ≤1 round; late verdicts are recorded, not enforced | Running every verdict synchronously | The main path slows down, then silently degrades into "no audit" | Partial | Q2 |
| INV-E6 | **Pre-checks must not become approvals**: pre-checks emit hints and risk, never "block everything" | Every action has to pass a gate | The unattended system stalls waiting for a human | Partial | Q2 |

---

## 3. Device (INV-D)

| ID | Invariant | Reverse-engineered from | Symptom when violated | Stance | Evidence |
|---|---|---|---|---|---|
| INV-D1 | **Judge liveness by artefacts, not by processes or dashboards** | A live process is not a live function | The process is alive, the service is dead, the light stays green | Running | Q1 |
| INV-D2 | **A health verdict may only be read from the end-of-chain artefact** | The observation point sits mid-chain | The second half gets killed while the panel is still green | Running | Q1 |
| INV-D3 | **Any executor the scheduler cannot observe must report its own outcome** | Scripts launched through a hidden-window launcher always exit 0 | The scheduler can never see a failure | Partial | Q1 |
| INV-D4 | **The zero-cost gate runs before any LLM**: signals pass deterministic rules first, then decide whether to wake anything | Every signal goes through a model | Cost scales with noise, and judgement gets impoverished | Running | Q1 (graded value is Q3) |
| INV-D5 | **Wake, do not poll**: cost follows events, not elapsed time | Fixed-interval polling | You pay during quiet periods and still react too slowly during incidents | Partial | Q2 |
| INV-D6 | **The action surface is deliberately narrow**: only idempotent actions self-heal, everything else escalates | Non-idempotent self-heal | Each repair makes it worse, and two repairs overwrite each other | Running | Q1 |
| INV-D7 | **Suppression granularity ≥ one independently failing unit, with TTL and audit trail** | Suppressing at a coarse granularity | One suppression blinds every real failure of that class for the whole TTL | Running | Q1 |
| INV-D8 | **Judge and deliver are separated**: the executor only writes the queue; the side that holds memory delivers (with quiet hours and dedup) | The executor pings a human directly | Duplicate storms, disturbed quiet hours, dropped messages | Running | Q1 |
| INV-D9 | **Three denoising layers, never a fourth**: task-level memory → suppression table (TTL) → time-window dedup | Adding a layer per false positive | Denoising itself becomes the most complex module | Partial | Q2 |
| INV-D10 | **Auxiliary sessions are created lazily**: absence is not evidence of failure | Reading "no session" as "patrol did not run" | Every misread triggers a pointless remediation | Running | Q1 |

---

## 4. Cross-cutting (INV-X)

| ID | Invariant | Reverse-engineered from | Symptom when violated | Stance | Evidence |
|---|---|---|---|---|---|
| INV-X1 | **Write authority of the supervisor and the supervised must be separated**: audit lines are never written by the task being audited | A task appends its own liveness line | When it fails, the liveness line itself goes blind | Running | Q1 |
| INV-X2 | **Modules couple through agreed files, not through calls** | Direct API calls between modules | Replacing either side forces changes on the other | Running | Q1 |
| INV-X3 | **Sessions are cache; files are truth**: every session may rotate daily, the main one included | Treating the session as storage | Clearing it loses memory | Running | Q1 |
| INV-X4 | **Free/local models for judgement, cheap models for generation** | Spending a premium model on judging | Cost becomes unpredictable and judging frequency gets cut | Running | Q1 |
| INV-X5 | **Silent failure must be distinguishable from silent success** | The "no output means fine" default | Failures stay unknown for weeks; incidents open with "it was working all along" | Running | Q1 |

---

## 5. Dependencies (violating one drags others down)

```
INV-X3 ──► INV-M4 ──► INV-M6          (sessions can rotate → settling must precede clearing; settling can have gaps → gaps must be reported)
INV-X1 ──► INV-D1 / INV-D2            (without write separation, neither liveness nor end-of-chain readings are trustworthy)
INV-X1 ──► INV-M9                     (without write/rate separation, the ledger loses discriminating power)
INV-D4 ──► INV-X4 ──► INV-E1          (a cheap gate is what makes a resident external judge affordable)
INV-M7 ──► INV-E4 ──► INV-M8          (refutable entries make real evidence necessary; trustworthy evidence makes blind review meaningful)
INV-X5 ──► INV-D3                     (demanding visible failure is what forces executors to report their own outcome)
```

**How to read this**: when the left side is violated, the invariants on the right still *formally* hold but are *effectively void*. That is exactly why failure modes are collected separately (front §5 plus one section per volume): **this class of failure raises no error.**

---

## 6. Change rules

1. A new invariant must state what it was reverse-engineered from. If you cannot, it is a preference, not an invariant → move it to the glossary or an appendix.
2. Deleting an invariant requires deleting every citation to it, or the references break.
3. A `claimed` entry that gets measured becomes `measured`; one that is falsified gets rewritten outright — **never quietly softened into vaguer wording**.
4. Published IDs never change (external references would break). Retired entries keep their ID, are marked `DEPRECATED`, and point to the replacement.
