# Appendix A: Reference Implementation Notes

> This appendix gives people who want to build it a sense of scale and boundaries. **Line counts are magnitudes, not promises**; swapping language, host, or storage changes them. To decide whether it applies to you, read the invariants, not the line counts.

- Code license: **MIT** (the rest of the document is CC BY 4.0)
- Total magnitude: **~3.0k lines**, spread over 8 subsystems that do not call each other

---

## A.1 Subsystems and magnitude

| # | Subsystem | Lines | Responsibility | Invariants |
|---|---|---|---|---|
| 1 | Session factory | ~400 | Create, adopt, inject opening content, archive | INV-X3 / INV-M4 |
| 2 | Tool gateway | ~250 | One routing table = capability list; dispatch by table; audit every call | INV-X2 / INV-E6 |
| 3 | Full-text search (CJK) | ~650 | Bigram indexing + wholesale rebuild + date-window queries | INV-M2 / INV-M3 / INV-M5 |
| 4 | Outcome reporting + scheduler fallback | ~200 | Executors self-report exit codes; fallback and timeout rescue on the scheduler side | INV-D3 / INV-X1 |
| 5 | Gate and escalation channel | ~800 | Three-state gate, suppression table, queue, delivery rules | INV-D4 / D7 / D8 |
| 6 | Nightly chain orchestration | ~300 | Three stages in series, gap reporting, no retry | INV-M6 |
| 7 | Index rebuild | ~250 | Rebuild derived index and question bank from the body | INV-M3 / INV-M5 |
| 8 | Judgment client | ~200 | Call an external model in a separate process, parse structured output, record | INV-E1 / INV-E2 / INV-E5 |

**How to read it**: 1–3 are memory, 4–5 are device, 6–8 are cross-domain glue. There is no "core big module" — each one can be thrown away and rewritten alone.

---

## A.2 The minimal interface of each subsystem (language-agnostic, shape only)

### 1 Session factory

```
create(date_key) -> session        # create a session, inject opening content
adopt(session, handover_text)      # adopt and inject yesterday's settling
archive(session) -> artefact_path  # archive into a compressed artefact
```

Constraint: `create` must be idempotent (a repeat call on the same day returns the existing session, never a new one).

### 2 Tool gateway

```
dispatch(action_name, params) -> {rc, stdout, artefacts[]}
list() -> [{module, func, desc, disabled}]
```

Constraint: `dispatch` accepts only names in the table; anything else returns non-zero rc and **is not executed**. Every call writes one audit line.

### 3 Full-text search (CJK)

```
index_rebuild(source_files) -> {docs, terms}
search(query, date_from, date_to, limit) -> [{doc, snippet, score}]
```

Constraint: `index_rebuild` must build from scratch (never merge with the old index); CJK requires bigram segmentation or a real tokenizer — **do not trust the default tokenizer**.

### 4 Outcome reporting and scheduler fallback

```
report(source, rc, note)           # called by the executor itself
watch(task_name) -> {last_result, last_run}
```

Constraint: the writer of `report` must be the executor; `last_result` read by `watch` is untrustworthy when the task is launched through a wrapper, so pair it with a fallback (timeout rescue / sentinel artefact).

### 5 Gate and escalation channel

```
gate(signal) -> {state: drop|defer|wake, reason}
enqueue(event)                     # append only
deliver(queue) -> {sent[], skipped[]}   # quiet hours, dedup, rate limit
```

Constraint: `gate` is a pure function with no side effects; `deliver` is the only entry point that disturbs a human.

### 6 Nightly chain orchestration

```
run_stages([extract, distill2, rotate]) -> {rc, gaps[]}
```

Constraint: any stage failure → record `gaps`, **do not retry**, return non-zero.

### 7 Index rebuild

```
rebuild(experience_body) -> {index, question_bank}   # no scores
```

Constraint: the emitted question bank **must not carry a confidence field** (halo-effect defence).

### 8 Judgment client

```
judge(snippet, rubric) -> {class, severity, anchor, reason}
```

Constraint: separate process / separate call; input is the raw snippet; an invalid output (missing reason or anchor) is recorded as invalid and must never fall back to "pass".

---

## A.3 Equivalents when swapping language or host

| Role here | Common equivalent |
|---|---|
| Session (conversational container with a tool surface) | Any agent framework session with tool calls |
| Scheduled tasks | OS scheduler / systemd timer / cron |
| Full-text search (FTS5 + bigram) | Any CJK-capable search library; otherwise implement bigram splitting yourself |
| Compressed archive | zstd / gzip / zip all fine |
| Structured ledger | JSONL appends are simplest (humans can read them too) |
| Judgment (external model) | Any callable model; cheap or local preferred |

**The one thing you cannot swap freely is the "files as interfaces" shape** — replace that and you have replaced the architecture, and two domains must be rewritten.

---

## A.4 What the reference implementation deliberately excludes

Honest list:

- **No** installer, configuration wizard, or container orchestration.
- **No** multi-user or permission model (single machine, single user assumed).
- **No** web UI (a dashboard is optional observation, never the source of a health verdict).
- **No** benchmark suite (the `claimed` entries have not been validated in a controlled experiment — see `07-sources.md`).

If you need any of the above, your scenario has left this document's assumptions; treat those as separate designs, not as missing features of this architecture.
