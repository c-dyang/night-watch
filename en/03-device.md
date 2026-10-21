# Device: Outcome Signals, the Zero-Cost Gate, and the Escalation Ladder

> This chapter answers three questions only: **how do we know something broke** (signals), **how do we avoid burning money on it** (the gate), **who is allowed to act** (the ladder).
> One line: **any outcome signal may wake an inspection, but it must first pass a gate that costs nothing.**

---

## 1. Every outcome signal

### 1.1 Four signal classes

| Class | Concrete form | Strong | Medium | Weak |
|---|---|---|---|---|
| **Liveness** | Heartbeat line, probe response, port reachable | Probe silent N times in a row | Silent once | Process absent |
| **Outcome** | Exit code, task return value, executor self-report | Non-zero exit | No return | Returned but semantically ambiguous |
| **Artefact** | File mtime / size / content, record count | Artefact missing or empty | Freshness overdue | File exists |
| **Scheduler** | Last result, next trigger time | Last result abnormal | Overdue to trigger | Status shows normal |

**Signal strength decides the processing level** (§3): strong signals may jump to L2/L4, weak ones only reach L0/L1.

### 1.2 Three rules for reading signals

1. **Judge liveness by artefacts, not processes** — a live process is not a live function [INV-D1].
2. **Health verdicts come only from the end-of-chain artefact** — an observation point mid-chain means you only observe the first half [INV-D2].
3. **The executor must report its own outcome** — any executor whose exit code cannot be observed (launched through a hidden-window launcher, wrapped in a script, invoked remotely) must write its own result line; otherwise the scheduler sees success forever [INV-D3].

### 1.3 Why "the dashboard is green" is the most dangerous state

A dashboard is **the observer's self-report**. Three situations look identical:

- It really is fine.
- The checker itself broke and emitted default values.
- The observation point is mid-chain and the second half is dead.

So a dashboard is an **index** ("probably fine — go check the end-of-chain artefact"), never a **conclusion**.

Corollary: **silent failure must be distinguishable from silent success** [INV-X5] — no observation point that can only emit "normal / no anomalies" satisfies this, because "no output" and "output says fine" are the same state there. This looks like a detail; it is actually where the whole chapter starts: **if dashboards could be trusted, no gate and no ladder would be needed.**

---

## 2. The zero-cost gate

### 2.1 Why a gate must come first

Order of magnitude:

```
signals : anomalies ≈ 100 : 1     (1000 : 1 in calm periods)
```

If every signal wakes a model:

- **Cost scales with noise**, not with anomalies;
- **Latency**: a model call is seconds, while "is this worth looking at" should be milliseconds;
- **Judgement fatigue**: the judge is fed mostly "nothing happened", so real anomalies stop standing out.

So the gate's job is not "decide severity" but **spend 99% of normal signals on deterministic rules and let only 1% reach anything that costs compute** [INV-D4].

### 2.2 Only four rule classes are allowed inside the gate

| Class | Form | Example (abstract) |
|---|---|---|
| **Freshness** | Compare artefact mtime with the expected period | "Daily report mtime is earlier than today's expected production time" |
| **Whitelist** | Is the target/action inside the allowed set | "This action belongs to the idempotent self-heal set" |
| **Dedup** | Fingerprint + time window | "Same source and same rule wakes at most once per 10 minutes" |
| **Known suppression** | Suppression table hit (with TTL) | "This (unit, rule) is suppressed, 3 days left" |

All four are table lookups or comparisons: no speculation, no causal attribution, no semantic understanding.

### 2.3 Three-state output, not two

| Output | Meaning | Downstream |
|---|---|---|
| **drop** | Clearly normal (freshness fine, outside whitelist and nothing to do) | Increment a counter, create no event |
| **defer** | Worth recording, not worth acting on now (weak signal, first occurrence, quiet hours) | Fold into the daily roll-up, no wake |
| **wake** | Worth one round of judging compute | Emit a wake event (C3) → execution domain |

**A two-state gate (act / don't act) is always wrong**: either noise reaches the model, or signals that need accumulation get dropped. The three-way split is "act now vs summarise today vs ignore".

### 2.4 Two design prohibitions for the gate

| Prohibition | Why |
|---|---|
| **The gate must have no side effects** | It must be a pure function: read signals, emit one of three states. Once it starts "fixing something along the way", a misjudgement becomes damage with no audit trail |
| **Business logic must not live in the gate** | Business rules drift; putting them in the gate means "to change business you change the gate", turning infrastructure into a junk drawer |

### 2.5 Error costs are asymmetric

| Error | Cost | Conclusion |
|---|---|---|
| **Miss** (should have woken, did not) | Silent rot, found weeks later | **Extremely high**, and invisible |
| **False wake** (woke when it should not) | One extra cheap judgement | Very low, and measurable |

Corollary: **thresholds should lean conservative (toward wake)**, and noise is suppressed by denoising rather than by raising the bar. This is the easiest place to get it backwards — the human instinct is "don't bother me", so the bar gets tuned to the point of silence, and silence is blindness.

### 2.6 Suppression granularity and TTL

> **Suppression granularity must be ≥ one independently failing unit, with a TTL and an audit trail.** [INV-D7]

| Granularity | Consequence |
|---|---|
| The entire "source" | One false positive blinds every real failure under that source for the whole TTL |
| **A single "target × rule"** | Blinds only the one thing actually believed to be a false positive |
| A single event | Nearly useless (event fingerprints differ every time); the suppression table balloons |

Three additional requirements: **TTL is mandatory** (open-ended suppression = permanent blindness); **the act of suppressing is itself recorded** (who, why, expiry); **expiry restores automatically**, rather than waiting for someone to remember.

> The most dangerous decision in practice is whether to suppress a whole category because "this one was a false positive". The answer is almost always no — tolerate one more false positive instead.

---

## 3. The escalation ladder: L0 → L4

### 3.1 Ladder definition

| Level | Name | Actor | Cost | Authority | Exit condition |
|---|---|---|---|---|---|
| **L0** | Silent record | Executor / probe | 0 | Write only | Always (every level passes L0 first) |
| **L1** | Daily roll-up | Nightly chain | 0 | Project only | End of day |
| **L2** | Wake inspection | Free/local model judgement | Very low | Judge, never change | Produces "worth acting on or not" |
| **L3** | Whitelisted self-heal | Idempotent action executor | Low | Whitelist only | Must self-verify after repair |
| **L4** | Escalate to human | Queue + delivery | Human | None | Human acts, or re-escalates on expiry |

**Every level passes through the one below it.** No channel jumps straight to L4, because jumping bypasses dedup and suppression and produces floods.

### 3.2 Escalation conditions (must be explicit)

- The same `(target, rule)` appears **N times** in the window ⇒ L2.
- A strong signal (missing artefact, non-zero exit) ⇒ straight to L2, no accumulation wait.
- L2 judges "worth acting on" and the action is whitelisted ⇒ L3.
- L3 self-verification fails / action not whitelisted / touches credentials or security surface ⇒ L4.
- L4 unhandled beyond the deadline ⇒ re-escalate (once per round, not an infinite loop).

### 3.3 L3 admission: idempotent actions only

> **Only idempotent actions may self-heal; everything else escalates.** [INV-D6]

Idempotence must be defined precisely: **running it twice and running it once yield the same result, with no accumulating side effects.**

| Action | Idempotent | Self-healable |
|---|---|---|
| Restart a stateless service | Yes (restarting twice still means "restarted") | Yes |
| Upsert one missing record | Yes (upsert by key) | Yes |
| Edit a config file | **No** (twice may stack or clobber human edits) | No |
| Delete a file / roll back data | **No** | Absolutely not |
| Notify a human | **No** (duplicate messages) | Not a heal action — a delivery action |

**Self-heal must self-verify**: after repairing, check the artefact again; a failed verification counts as "not repaired" and escalates, not as "attempted, close it".

### 3.4 Judge and deliver are separated

> **The executor only writes the queue; delivery belongs to the side that holds memory.** [INV-D8]

| Role | Responsibility | Why |
|---|---|---|
| Judge (L2) | Judge + write one queue record | It holds no memory, so it cannot know "how many times we bothered someone today" |
| Executor (L3) | Repair + self-verify + write one queue record | Same reason |
| Delivery side | Read queue → deliver under quiet hours, dedup, rate limits | **It holds memory**, so it can decide "should this be said now" |

Typical symptoms of merging all three: **the same problem reported eight times in an hour**, or a call at 3 a.m. Delivery rules (queue-only during quiet hours, at most one delivery per class per 12 hours, at most 3 per round) are inherently memory judgements.

### 3.5 Three denoising layers, never a fourth

```
1. Task-level memory: the producer remembers "I just reported this"
2. Suppression table (with TTL): entries judged false-positive or known do not wake again
3. Time-window dedup: one wake per fingerprint per window
```

[INV-D9] Why a fourth is forbidden: each new layer exists to fix one specific false positive, and layers then mask each other — you end up unable to say why a signal was swallowed or which layer to change. **Make a layer coarser instead of adding another.**

### 3.6 Auxiliary sessions are created lazily

An absent patrol session is not a failed patrol [INV-D10].

| Observation | Wrong conclusion | Correct test |
|---|---|---|
| No patrol session recorded today | "Patrol did not run" | Check the **artefact** (today's day log / audit line) |
| Session created but empty | "Patrol ran and found nothing" | Same: check the artefact, not the session |

A session is an **attention container**, created on demand. Treating "is the container there" as a health metric is another instance of using the dashboard as the conclusion.

---

## 4. Minimal contract

### 4.1 Rule table (one measurable assertion per row)

```json
{
  "id": "R-001",
  "target": "<observed object: service / task / file / queue>",
  "signal": "liveness | outcome | artefact | scheduler",
  "predicate": "<measurable assertion including threshold>",
  "severity": "low | medium | high",
  "evidence": ["<which field must be recorded as evidence>"],
  "action_class": "none | readonly | idempotent_whitelist | escalate",
  "ttl_days": 14
}
```

Requirement: **`predicate` must be decidable by a script** (threshold + field). "Looks healthy" is not a predicate.

### 4.2 Reporting contract (who reports what)

| Who | Reports | Form |
|---|---|---|
| Executor (unobservable by the scheduler) | Its own exit code and key results | One structured line, written **by the executor itself** |
| Supervisor (external checker) | Whether the artefact appeared / is fresh | One audit line, **never** written by the task being checked [INV-X1] |
| Scheduler | Last result and next trigger time | Provided by the scheduler, used as supporting evidence only |

### 4.3 Wake event (C3)

```json
{
  "signal_id": "<fingerprint>",
  "rule_id": "R-001",
  "raw": {"<verbatim signal, unprocessed>"},
  "gate": {"state": "wake", "reason": "<why it passed>", "dedup_age_s": 0},
  "at": "<ISO timestamp>"
}
```

Key point: `raw` must carry the **unprocessed original**, because the downstream judge must form its own conclusion rather than consuming a second-hand "looks like a failure". This is [INV-E2] applied to the device domain.

### 4.4 Three measurable checks

1. **Injection test**: create a real failure on purpose (e.g. stop an artefact from updating) and confirm it reaches L2 within the expected time; then make a whitelisted action fail and confirm it reaches L4. **An inspection that was never injection-tested is no inspection.**
2. **Zero cost in calm periods**: on a fully normal day, the number of model calls should be **0** (only the gate runs). Non-zero means the gate failed or judging sank into it.
3. **Suppression is auditable**: pick any suppression record and answer "who, when, why, expiry".

---

## 5. Failure modes in this chapter (honest list)

| Failure | Symptom | Structural cause |
|---|---|---|
| Green dashboard, dead chain | Everything is fine until it turns out nothing ran for three weeks | Observation point mid-chain [INV-D2] |
| The scheduler always sees success | Failures exist only in logs | Executor did not self-report [INV-D3] |
| Suppression as blinding | One false positive buys long silence | Granularity too coarse, no TTL [INV-D7] |
| Self-heal makes it worse | Running an action twice stacks | Non-idempotent action entered the whitelist [INV-D6] |
| Escalation as harassment | Same problem repeatedly, or a 3 a.m. call | Judge and delivery not separated [INV-D8] |
| Absent patrol treated as failure | A remediation fires with nothing wrong | Container used as health metric [INV-D10] |
| The gate grows into a business system | Changing business means changing the gate | Side effects and business logic entered the gate |
| Thresholds tuned to "just quiet" | Long silence, then a backlog of traceable misses | Raising the bar instead of denoising [INV-D4] |

---

## 6. Building the smallest version (order matters)

1. **Make executors self-report first**: any executor whose outcome is unobservable must write one line with its exit code and key results. **Without this step everything after it is built on illusion** [INV-D3].
2. Write **three rules** (one liveness, one outcome, one artefact freshness), containing nothing but thresholds and fields.
3. Write the **gate**: three-state output, pure function, no side effects; implement freshness and dedup first.
4. Write **event persistence + the queue**: wake events appended one at a time; the queue is a file for humans.
5. Write the **delivery side**: quiet hours, per-class dedup, per-round cap — those three rules are enough, do not build more.
6. Add **L3 whitelist**: exactly one idempotent action, and it must self-verify.
7. Run **one injection test**: create a failure and watch it travel the whole path.

By step 4 you "know something broke"; step 6 is where self-healing begins; step 7 is where credibility begins. **An inspection that skips step 7 is only psychological comfort.**
