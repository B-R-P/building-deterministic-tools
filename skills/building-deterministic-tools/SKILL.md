---
name: building-deterministic-tools
description: >
  Use when a task's correctness depends on faithfully consuming a large
  structured source — design exports, generated markup, long logs, data dumps —
  where reading by hand is slow, a missed detail breaks the outcome, or the
  same source must be queried repeatedly. Build a small deterministic CLI tool
  and query it on demand instead of re-reading the raw file. Do NOT activate
  when the source is small and single-use, a maintained parser/analyzer already
  exists, or nothing about the outcome depends on a missed detail.
---

# Building Deterministic Tools

## Overview

An AI agent is probabilistic. Reading 5000 lines of generated markup by eye
and *hoping* nothing is missed is a bet, not a plan. Any task that depends on
faithfully consuming a large structured source has a **determinism gap**
— close it with a small deterministic tool that parses the source once, then
answers the agent's questions about it one at a time.

**The spine: determinism makes you repeatable; reachable-and-verified make you
right.**

- **Deterministic** — same input, same output, every run.
- **Reachable (or loud)** — every input element can be pulled through a
  command, or is flagged loudly as unresolved.
- **Verified** — output is checked by a route other than the one that produced
  it, before trust.

Determinism without reachable-and-verified replaces probabilistic misses with
deterministic lies: confident, repeatable, wrong. The extractor that ran clean,
produced a tidy index, and silently dropped data. Or the friendly-looking tool
that cannot reach the one element the task needs.

The reader of the output is also probabilistic: the answers feed a later
session whose belief is a next-token prediction. Evidence must be re-derivable
from the source, not believed.

## When NOT to Use

- Source is small enough to read fully in one pass and used once.
- A maintained parser/analyzer for the source already exists.
- The output is consumed only by this session and never verified against the
  source.
- The outcome does not depend on missing any detail.

---

## Phase 1: Decide — build the tool or read by hand

**Entry:** The task involves consuming a structured source.

**Exit:** A verdict: `build` or `eyeball`.

**Steps:**

1. Measure the source: count layers/rows/elements. Large means hundreds or
   thousands of repetitive, machine-generated items.
2. Assess the stakes: does a missed detail break the outcome? Listen for the
   user's own words — "don't miss any detail", "every X must be preserved".
3. Assess reuse: will the output feed later sessions, a redesign, or a
   verification against the source?
4. Verdict: `build` if any of (large, high-stakes, reused). Otherwise `eyeball`.
5. Record the verdict and the measured count — it seeds the Phase 3 gate's
   independent count.

**Failure behavior:** when borderline, choose `build` if the stakes are
correctness; the tool is cheap, the missed detail is not.

---

## Phase 2: Build the tool — deterministic and agent-friendly

**Entry:** Verdict is `build`.

**Exit:** The tool parses the source once, exposes an overview that shows what
it can answer, and answers each question through a retrieval path that returns
a complete, untruncated element. Every element is reachable from the overview.
Re-running produces byte-identical output.

**Steps:**

1. Parse with a real parser (`html.parser.HTMLParser`, `json`, `csv`, etc.) —
   never hand-rolled string slicing.
2. Record **every** attribute and style key on every element; buffer raw
   embedded markup (e.g. full `<svg>` for icons) verbatim.
3. Resolve identifiers by **name**, not by value. A token reference like
   `var(--ink-ink, ...)` matches its app counterpart on the token NAME because
   the two sides store values in different spaces (app `oklch()` vs design hex
   fallback). Matching on the hex value fails silently.
4. The raw dump is complete — render every style key, filter nothing.
   Filtering is where data dies.
5. Report unresolved items as a separate, explicit list (unmatched vars, raw
   colors). They are decisions for the consumer, not noise.
6. Name output per source (`<slug>-dump.json`, ...) so multiple inputs never
   collide.
7. Derive everything downstream — the overview and every answer — **from the
   raw dump**, never by re-parsing the source. A bug fix then re-derives in
   seconds and a re-run never touches the source a second time.
8. **Design the interface for the consumer.** An agent can only use what it
   can find and reach:
   - An **overview** — cheap, small, answers "what is here and how many":
     element counts, distinct keys, the names a query can target. The agent
     reads the overview first; it must fit in one glance. It doubles as the
     digest and later carries the verification receipt.
   - A **retrieval path for every element type** — one question per call.
     Querying one element returns that element complete: intact text, resolved
     tokens, clean encoding — never summarized, never truncated.
   - **No dead ends** — walk from the overview to any element by composing its
     commands. What the overview names, a command can pull.
   The exact command shape is the agent's call. The contract is: discoverable,
   one question per call, complete answers, everything reachable.
9. **Keep every call small.** Prefer many cheap, bounded invocations over one
   that prints everything. Where a set is large, paginate or slice rather than
   dump.

**Failure behavior:** if a parse step throws, print the offending input to
stderr and skip that file — never emit a partial tree as if it were complete.

---

## Phase 3: Verify, then park the tool

Verify by a route other than the one that produced the answer. The code path
that emitted the output must never be the path that certifies it — a
self-derived count only proves the tool agrees with itself. Four cheap checks,
each with a distinct job:

**Entry:** The tool answers through its interface.

**Exit:** All four checks pass; the overview carries the receipt; the script is
parked; the actual task resumes against the tool.

**Steps:**

1. **Re-run, diff — catches nondeterminism.** Run the tool unchanged and diff
   the raw output against the previous run. Any difference that no source
   change explains means the tool is nondeterministic (set/dict ordering,
   timestamps). Fix and re-run.
2. **Independent count — catches quantity loss.** Count elements against the
   *source itself* by a second method (e.g. `grep -c` on the raw file), not
   from the tool's reported count. A mismatch means the extractor dropped
   whole items. This is the check that catches a flush-buffer bug: re-running
   the tool cannot, because a deterministic bug reproduces every run.
3. **Spot-check through a retrieval path — catches quality loss.** Pick 2–3
   known elements from the source and pull each *through the interface
   command that claims to reach it*. Confirm the answer is complete: correct
   resolved token, intact text, clean encoding. Counts cannot see per-element
   key loss — this is the load-bearing check, because a layer count can match
   while every layer silently lost its `outline` and `border-left`.
4. **Reachability sweep — catches dead ends.** The overview names element
   classes; the independent count sizes them. For each class, confirm a
   retrieval path exists and return one sampled element complete. If a class
   has no path, the tool cannot be trusted to answer the task.
5. Only when all four pass, append the receipt to the overview: `verified:
   idempotent, count = N, spot-check = 3/3`. Then consume the tool.
6. Leave the script in `tools/` (or the scratch dir) — do not wire it into the
   product, the build, or CI.
7. Do not gold-plate: these checks *are* the test — no test files, no
   committed fixtures, no generalization, no config surface. A fixture is just
   input the extractor already handled, so it cannot catch what the extractor
   cannot see.
8. Resume the real task by querying the tool. Do not re-parse the raw source
   by hand "to be sure" — that re-introduces the miss risk the tool removed.

**Failure behavior:** never consume unverified output. Any red check means the
tool is wrong, and the task must wait for the fix.

---

## Anti-Rationalization Table

| Rationalization | Redirect |
|---|---|
| "This file is small, I can read it by hand" | Small is fine to eyeball — but measure first. The moment it is large, high-stakes, or reused, build the tool. |
| "I know the output is right, I saw it" | **Hard stop.** Seeing is probabilistic; a re-run against the known-good count is deterministic. Run the gate. |
| "The tool's count proved the output is complete" | **Hard stop.** Self-reported verification. The count only proves the tool agrees with itself — re-derive from the source with an independent count. |
| "I'll just grep the source when I need a value" | Grep per value is slow and re-introduces the miss risk. The overview names it; one retrieval call returns it whole. |
| "The values didn't line up, I'll match by the raw hex" | The two sides store values in different spaces (oklch() vs hex). Resolve by name; report unmatched as decisions. |
| "This tool is useful, let me wire it into the repo properly" | The skill is about the *analysis* being deterministic, not the tool being permanent. Park it; resume the task. |
| "I dropped those attributes — they weren't important" | **Hard stop.** Reachable or loud. If it isn't in the dump, it is lost. Render everything or flag it. |
| "The counts matched, so nothing was dropped" | Counts cannot see per-element key loss. Spot-check a retrieval path; reachable or loud. |
| "One command that prints everything is simpler" | A single fat call fills the context and buries the detail the task needs. Slice it into an overview plus targeted paths. |
| "That element class doesn't need a path, the task won't ask" | Reachability is the contract. Every class the overview names must have a working path; the consumer decides what it asks for. |