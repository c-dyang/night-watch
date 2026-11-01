# Change Governance · From Script Headers to an Admission Gate

> This volume answers one question: **I changed this — what else did I forget to change?**
> It does not judge whether the change is right. It only answers "who else depends on it" — which, in a personal environment, is the usual shape of an incident.

- Version: `v0.1-draft` ｜ Author: `c-dyang` ｜ Docs license CC BY 4.0

---

## 1. Shape of the problem: a personal environment has no consumer chain

The nightly chain is **file flow**: something writes `nightly/<date>.md`, something else reads it to build `review/<date>.md`.

In a company that chain is forced into view — other people read your output, so the format *is* a protocol, and breaking it gets you yelled at. In a personal environment the only consumer is you: **both ends of the contract are the same person**, so no external pressure ever enumerates "who reads me". Every change therefore carries a blast radius that fails silently.

Two real cases, seven weeks apart, same failure mode:

| Date | Change | What was missed | Symptom |
|---|---|---|---|
| 2026-09-09 | stdout format of `memory_fts_index.py` | two parsing defects in `fragment_search.py` | search results landed in the wrong fields |
| 2026-10-30 | added a `⟨seq N⟩` prefix to nightly skeleton lines | `nightly_review.py:113` anchors `^(?:\[HH:MM\])?\[(用户\|助手)\]` at `^` | hits 199/228/224 → **0/0/0**; review kept only webhook logs; **conversation blind for a day**, spotted by a human at 22:4x only because the file was 876 B |

Shared signature: **only the new consumer was verified, never the existing ones**, and the failure is **silent zero** — no error, just hits dropping to 0. A total blackout shows up immediately; this kind survives a whole day.

## 2. Why the first design was wrong: it built a contract where a hint was needed

V1 gave every script `format: <token> | <regex>` and cross-checked consumers by format anchor. One run hit three semantic traps:

1. **Ambiguity**: is `format` "the format I produce" or "the format I depend on"?
2. **Placeholder runaway**: `-` meant "no output", and a literal scan matched everything.
3. **Upstream false hits**: producers naturally mention their consumers' tokens, in the wrong direction — without an ancestor exemption every chain reports false positives.

All three are fixable, but the cost is **keeping the alarm at zero false positives forever** — one false positive and people learn to ignore it, and the whole thing is dead. So the conclusion inverted:

> **Nobody asked for a contract; they asked for "who else".** Drop the format field, ask "who reads the path I write", and all three traps vanish.

## 3. Final form: five address-only fields

```python
# [MANIFEST v1]
# id:        nwfcmpxt
# script:    scripts/nightly/distill_extract.py
# reads:     ~/.dsh/sessions/<ws>/session*.jsonl.zstd ; .../distill_watermark.json
# writes:    qq-personal-assistant/nightly/<date>.md
# downstream:vswzou5g
# [/MANIFEST v1]
```

Four hard rules:

- **Downstream references `id`, never a path** — so renaming or moving a script does not break the graph. This is the load-bearing decision.
- **One value per line, no inline comments, no continuations** — an inline `←` note once produced three dangling references.
- **The header holds addresses, not descriptions** — prose belongs in the docstring. Function is already in the addresses (which files it reads and writes).
- **Versioned block marker** `[MANIFEST v1]` — an unversioned marker silently matched nothing and the tool ran over an empty graph.

## 4. Identity: the birthplace rule

`id` is neither random nor a content hash. It is derived from **where the script stood when first registered**:

```
id = base32_lower(sha1(norm(first-registered path)))[:8]
```

Three responsibilities, never mixed:

| Concept | Carrier | Changes? | Purpose |
|---|---|---|---|
| Birthplace | **`id`** | never | **identity** (graph key) |
| Current address | `script` | yes | locating + MOVED detection |
| Content fingerprint | **not in the header** | every edit | integrity (git / backups' job) |

**Identity must be orthogonal to content.** Use an md5 as an id and a one-character edit mints a new identity — the graph breaks daily. A hash is a content fingerprint, not an identity.

A derived value (rather than a random string) buys **replayability**:

- Hit an unlabelled script, compute `f(current path)` and compare with the registry → a match means "it never moved, it only lost its tag" → restore the same identity.
- One path can only ever produce one id ⇒ registration is **idempotent**.
- Both kinds of dangling converge: **record without file** = DEAD (deletion detector; never auto-deleted); **file without record** = ORPHAN (compute `f`, match and re-tag).
- After a move, `id` no longer equals `f(script)` — not an error, but a fossil of "where it used to be", read out by the MOVED alarm.

### Four tiers of kinship evidence

| Strength | Evidence | Action |
|---|---|---|
| 1 | same place: `f(path)` equal | auto-adopt |
| 2 | functional fingerprint: `reads`/`writes` overlap | auto-suggest |
| 3 | neighbourhood signature: Jaccard of upstream ∪ downstream | flag for confirmation |
| 4 | basename collision | list only |
| — | none | treat as a **new node**, mark the old DEAD, **never guess** |

Output overlap implies neighbourhood overlap but not the reverse ⇒ the natural ordering is place > output > neighbourhood > name. **Move + rebirth + new output path** together are unrecognisable — a known blind spot, stated rather than hidden.

## 5. The graph is emergent; there are only four checks

No central registry is needed: `downstream` cites another script's `id`, so the dependency graph grows by itself (equivalent to one `import` line per script).

| # | Check | Kind |
|---|---|---|
| 1 | dangling reference | static |
| 2 | chain head (no inbound edge) | informational |
| 3 | cycle (two chains rewriting the same artifact) | static |
| 4 | **path-level diff**: scripts whose `reads` hit my `writes` minus my declared downstream | static |
| 5 | blast radius: transitive downstream closure = regression list | static |

**Only check 4 speaks up when I forgot to declare someone.**

One earlier idea was dropped: an "entry whitelist" to detect unreachable nodes. A whitelist is a second source of truth and must drift, and on the graph "unreachable" and "chain head" are indistinguishable — hard-judging it guarantees false positives. So the whitelist went away; chain heads are derived from having no inbound edge, and missed declarations are check 4's job.

## 6. Launchers sit at the end of the chain and get no header

`.bat` / `.ps1` launchers **do not carry a header**, for three stacked reasons:

1. **Encoding minefield**: GBK + no BOM + CRLF, while our headers contain `<date>` placeholders — `cmd` always expands `%` inside `REM` and treats `<`/`>` with risk.
2. **Second source of truth**: the body already says `"%PY%" ...nightly_archive.py`; restating it as `downstream` stores one fact twice, and a rename will update only one copy.
3. **The relation is not a data edge**: a launcher produces and consumes nothing; it only answers "who pulls it up".

So we **extract from the body** instead: only non-comment `.py` invocations (historical steps mentioned in comments are listed but create no edge), stripping `%~dp0` / `%VAR%` / `$PSScriptRoot`, with edges pointing `script → launcher` (a **sink**). The `id` is computed each run and never stored — the legitimate use of the birthplace rule: **because it is replayable, it need not be written down.**

A free alarm comes with it: **a launcher mentions a node the graph does not have** ⇒ the script was renamed and the launcher was not.

## 7. The admission gate: three checks, all mandatory

A chain inventory only has force once it becomes a gate. Any change to a **production script**:

1. **Before**: obtain `id` + regression list (downstream closure). **Blast radius ≥3 or cross-chain ⇒ report first, split the work.**
2. **After**: format lint clean (no ERR / MOVED / DEAD / DUPLICATE) **and** an empty diff — "someone reads my output but I did not declare it" must be fixed before proceeding.
3. **Verify**: `py_compile` **plus a real run**; if the output format or a line-start anchor changed, **replay against real historical files** and the hit count **must not go from N to 0**.

One-sentence criterion: **a verification that can run automatically may not be replaced by "I read it once".** Blast radius is itself the risk number — one downstream, change it; three or more, speak first.

## 8. Real output: three edges that only the diff could surface

After labelling 15 nodes inside the chain, the checker found three dependencies **nobody remembered**:

| Edge | Content | Why it was forgotten |
|---|---|---|
| `nightly_archive → session_bridge` | archiving also moves `review-<date>.md`, and `session_bridge` reads the archived fallback path | it spans two phases — archiving and injection |
| `pending_archive_nightly → memory_fts_index` | the first writes `USER.md`, the second indexes `USER.md` | changing profile format must rebuild the index |
| `external_audit → experience_index` | the first writes `pending_verdicts.json`, the second reads and applies it | **the checker independently reproduced the ③→④ order documented in the `.bat` comments** |

The third is the most convincing: the graph was derived from read/write paths alone, yet it agrees with the hand-written chain order. **The tool is checking the code, not reciting my memory.**

## 9. Boundaries (written down so they are not "fixed" as bugs later)

1. **Convention-level dependencies are invisible**: when the consumer reads a "line prefix / numbering convention" rather than a file (`resolve_seq` reads the archive, not `nightly/`), there is no path intersection — the edge must be added by hand.
2. **Leaf kinship is weak**: a script without `writes` keeps only upstream as its fingerprint.
3. **The graph never completes itself**: the machine guarantees your declared edges are self-consistent and catches part of the leakage via paths; the rest is human work. Its value is **a live list**, not **a complete list**.
4. **The registry is derived**: snapshot and graph rebuild from headers; the table may hold nothing unique.
5. **Dynamic paths leak**: unusual names built by `os.path.join` defeat path matching; tightening that means runtime observation at double the cost.

## 10. Invariants referenced by this volume

- **[INV-X6]** Declarations and facts must be reconciled in both directions — a one-way self-report is never complete, and it fails silently.
- **[INV-X7]** Identity is orthogonal to content and must be replayable — otherwise deletion, missing tags and moves all end up as manual work.
- **[INV-E7]** The admission gate's inputs (blast radius) and its verification must be machine-produced — confirmation covers intent, never the checklist.

> One line to close: **a company turns "forgot" into "cannot commit"; a personal setup can only turn "forgot" into "known by next run".**
> The first needs an OA stack. The second needs a graph you can recompute every time.

---

[Back to contents](../README.md)
