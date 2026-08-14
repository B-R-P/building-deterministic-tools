---
name: building-deterministic-tools
description: >
  Use when a task's correctness depends on faithfully consuming a large
  structured source — design exports, generated markup, long logs, data dumps —
  where reading by hand is slow or a missed detail breaks the outcome, or when
  a parsed result must be trusted and reused by later sessions. Do NOT activate
  when the source is small and single-use, or a maintained parser already
  exists.
---

# Building Deterministic Tools

## Overview

An AI agent is probabilistic. Reading 5000 lines of generated markup by eye
and *hoping* nothing is missed is a bet, not a plan. Any task that depends on
faithfully consuming a large structured source has a **determinism gap**
— close it with a small deterministic tool that parses the source once and
emits artifacts you Read/Grep instead of re-parsing the raw file.

**The spine: determinism makes you repeatable; lossless and verified make you
right.**

- **Deterministic** — same input, same output, every run.
- **Lossless (or loud)** — every input element lands in the output, or is
  flagged loudly.
- **Verified** — output is checked by a route other than the one that produced
  it, before trust.

Determinism without lossless-and-verified replaces probabilistic misses with
deterministic lies: confident, repeatable, wrong. That is the failure mode
this skill exists to prevent — the extractor that ran clean, produced tidy
output, and silently dropped data.

The reader of the output is also probabilistic: the artifacts feed a later
session whose belief is a next-token prediction. Evidence must be re-derivable
from the source, not believed — and it must never be certified by the same
code path that produced it.

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

## Phase 2: Build the tool — lossless and agent-readable

**Entry:** Verdict is `build`.

**Exit:** The script parses the source and produces a full artifact set plus a
short digest that points back to the full files. The tool is re-runnable:
same input produces byte-identical output.

**Steps:**

1. Parse with a real parser (`html.parser.HTMLParser`, `json`, `csv`, etc.) —
   never hand-rolled string slicing.
2. Record **every** attribute and style key on every element; buffer raw
   embedded markup (e.g. full `<svg>` for icons) verbatim.
3. Resolve identifiers by **name**, not by value. A token reference like
   `var(--ink-ink, ...)` matches its app counterpart on the token NAME because
   the two sides store values in different spaces (app `oklch()` vs design hex
   fallback). Matching on the hex value fails silently.
4. Render every style key into the tree output — do not filter to a "useful"
   subset. Filtering is where data dies.
5. Report unresolved items as a separate, explicit list (unmatched vars, raw
   colors). They are decisions for the consumer, not noise.
6. Name artifacts per-source (`<slug>-design.json`, `-tree.txt`, `-tokens.css`,
   `-icons.txt`) so multiple inputs never collide.
7. Derive everything downstream — tree, tokens, digest — **from the
   `-design.json` dump**, never by re-parsing the source. A bug fix then
   re-derives in seconds and a re-run never touches the source a second time.
8. Emit a short digest.md and decisions.json that summarize the skeleton,
   unresolved items, and counts, point to the full files, record the source
   file they came from, and cite the resolved decisions. After verification,
   append the receipt (Phase 3 step 4). The digest stays lossless by
   reference — readable at a glance, full files as source of truth.

**Failure behavior:** if a parse step throws, print the offending input to
stderr and skip that file — never emit a partial tree as if it were complete.

---

## Phase 3: Verify, then park the tool

Verify by a route other than the one that produced the answer. The code path
that emitted the output must never be the path that certifies it — a
self-derived count only proves the tool agrees with itself. Three cheap checks,
each with a distinct job:

**Entry:** The artifact set is written.

**Exit:** All three checks pass; the digest carries the receipt; the script is
parked; the actual task resumes against the artifacts.

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
3. **Spot-check — catches quality loss.** Pick 2–3 known elements from the
   source and confirm they land in the output with the correct resolved token,
   intact text, clean encoding. Counts cannot see per-element key loss — this
   is the load-bearing check, because a layer count can match while every
   layer silently lost its `outline` and `border-left`.
4. Only when all three pass, append the receipt to the digest: `verified:
   idempotent, count = N, spot-check = 3/3`. Then consume the artifacts.
5. Leave the script in `tools/` (or the scratch dir) — do not wire it into the
   product, the build, or CI.
6. Do not gold-plate: these checks *are* the test — no test files, no
   committed fixtures, no generalization, no config surface. A fixture is just
   input the extractor already handled, so it cannot catch what the extractor
   cannot see.
7. Resume the real task by reading the artifacts. Do not re-parse the raw
   source by hand "to be sure" — that re-introduces the miss risk the tool
   removed.

**Failure behavior:** never consume unverified output. Any red check means the
extractor is wrong, and the task must wait for the fix.

---

## Anti-Rationalization Table

| Rationalization | Redirect |
|---|---|
| "This file is small, I can read it by hand" | Small is fine to eyeball — but measure first. The moment it is large, high-stakes, or reused, build the tool. |
| "I know the output is right, I saw it" | **Hard stop.** Seeing is probabilistic; a re-run against the known-good count is deterministic. Run the gate. |
| "The tool's count proved the output is complete" | **Hard stop.** Self-reported verification. The count only proves the tool agrees with itself — re-derive from the source with an independent count. |
| "I'll just grep the source when I need a value" | Grep per value is slow and re-introduces the miss risk. The artifacts answer in one Read. |
| "The values didn't line up, I'll match by the raw hex" | The two sides store values in different spaces (oklch() vs hex). Resolve by name; report unmatched as decisions. |
| "This tool is useful, let me wire it into the repo properly" | The skill is about the *analysis* being deterministic, not the tool being permanent. Park it; resume the task. |
| "I dropped those attributes — they weren't important" | **Hard stop.** Lossless or loud. If it isn't in the output, it is lost. Render everything or flag it. |
| "The counts matched, so nothing was dropped" | Counts cannot see per-element key loss. Spot-check known elements; lossless or loud. |