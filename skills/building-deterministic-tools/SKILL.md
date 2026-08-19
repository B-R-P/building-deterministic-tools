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
5. Record the verdict and the measured count — it seeds the Phase 4 gate's
   independent count.

**Failure behavior:** when borderline, choose `build` if the stakes are
correctness; the tool is cheap, the missed detail is not.

---

## Phase 2: Choose reusable vs throwaway

**Entry:** Verdict is `build`.

**Exit:** A lifecycle choice: `reusable` or `throwaway`, made against the single
criterion below. This criterion is the only source of the reusable/throwaway
rule — Phase 4 refers to the two paths without re-deriving the choice.

**The lifecycle criterion:**

- **Reusable** — the source shape is generic: markup, logs, exports, dumps —
  the same kind recurs across sessions or sources. Choose this by default; the
  build cost is paid once and reused.
- **Throwaway** — the source shape is provably one-off: a specific input that
  will never recur. Use the lean park-in-scratch path.
- Choose `reusable` unless the one-off case is established. A reusable tool's
  cost is paid once and reused; a throwaway tool's cost is paid every time the
  shape recurs.

**Failure behavior:** when the shape's recurrence is unclear, choose `reusable`
— the persistence step is cheap, and re-parsing a recurring source is not.

---

## Phase 3: Build the tool — deterministic and agent-friendly

**Entry:** Verdict is `build`; lifecycle is chosen.

**Exit:** The tool parses the source once, exposes the fixed documentation core
(overview, catalog, help), and answers each question through a retrieval path
that returns a complete, untruncated element. Every element is reachable from
the overview. Re-running produces byte-identical output.

**Steps — parse and dump:**

1. Parse with a real parser (`html.parser.HTMLParser`, `json`, `csv`, etc.) —
   hand-rolled string slicing is a miss risk, not a shortcut.
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

**Steps — the fixed documentation core** (every tool has all three; this shape
is what any agent can rely on):

- `<tool> overview` — one-glance digest: counts, distinct keys, queryable
  names, + verification receipt. The agent reads this first; it must fit in one
  glance.
- `<tool> catalog` — self-description: list of subcommands, each with
  description, param schema (types/valid values/examples), and what it returns.
  New agents learn the whole surface from the tool itself.
- `<tool> help <subcommand>` — per-command detail.

**Steps — scenario-built retrieval commands** (free-form, built per source, one
command per element type / query pattern). Each follows MCP-inspired discipline,
adapted for agents:

- verb-first names, service-prefixed to avoid collisions
- description answers *what / when-to-use / what-it-returns* (+ "when NOT to
  use")
- params documented with valid-value examples and format hints
- errors that teach the fix (e.g. `date` must be ISO 8601, e.g. 2026-01-15; got
  'next Thursday')
- responses return complete-but-lean data (summaries + drill-down, pagination
  with `has_more` / `total`)
- one question per call; many cheap, bounded invocations over one that prints
  everything; paginate or slice large sets rather than dump

**The behavior contract:** reachable-or-loud, one question per call, no dead
ends, byte-identical re-runs. What the overview names, a command can pull —
walk from the overview to any element by composing its commands.

**Failure behavior:** if a parse step throws, print the offending input to
stderr and skip that file — never emit a partial tree as if it were complete.

---

## Phase 4: Verify, then persist

Verify by a route other than the one that produced the answer. The code path
that emitted the output must never be the path that certifies it — a
self-derived count only proves the tool agrees with itself.

**Entry:** The tool answers through its interface.

**Exit:** The checklist below is fully checked; the tool is persisted along the
Phase 2 path; the actual task resumes against the tool.

**The verification checklist:**

```
☐ Re-run diff = zero delta
☐ Independent element count = source count
☐ 3+ spot-checks resolve by name
☐ Reachability sweep: every cataloged element queryable
☐ Tool has catalog + overview + help
☐ Reusable tools have AGENTS.md pointer
```

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
5. **Documentation core — catches invisibility.** Confirm the tool answers
   `overview`, `catalog`, and `help <subcommand>`; a tool that cannot describe
   itself is invisible to later sessions.
6. **Persistence — catches orphaned tools.** On the reusable path, confirm the
   AGENTS.md pointer exists (name, location, invocation, what it answers,
   trigger description). On the throwaway path, confirm the tool stays parked
   and is discarded after use.
7. Only when every item checks, append the receipt to the overview: `verified:
   idempotent, count = N, spot-check = 3/3`. Then consume the tool.

**Persist along the Phase 2 path:**

- **Reusable:** persist under `skills/building-deterministic-tools/tools/<slug>/`
  (CLI + dump + catalog). Register the tool in the repo's `AGENTS.md` — add one
  entry under `## Built tools`:

  ```markdown
  - `<tool>` — `<location>` — invoke with `<command>` — answers `<what>` —
    query when `<trigger description>`
  ```

  The **trigger description** tells later sessions when to query this tool
  instead of re-parsing. The tool stays out of the product, the build, and CI.
  The checks are the test — no test files, no committed fixtures, no
  generalization, no config surface. A fixture is just input the extractor
  already handled, so it cannot catch what the extractor cannot see.
- **Throwaway:** park it in the scratch dir, keep it lean, discard after the
  task completes.

**Consume:** query the tool for every answer. Re-parsing the raw source by hand
re-introduces the miss risk the tool removed; evidence comes from the tool, and
the tool is verified.

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
| "I'll skip the catalog, the overview says enough" | **Hard stop.** A tool without a catalog is invisible: new agents can't learn its surface. The catalog is the self-description; the overview is the digest. |
| "That element class isn't reachable, but I can read it by hand" | **Hard stop.** Reachable or loud is the contract. If a class has no path, the tool is wrong — fix the tool, and the task waits. |
| "This source might recur, but the throwaway path is faster" | Choose reusable unless the one-off case is established. A reusable tool's cost is paid once; re-parsing a recurring source is paid every time. |
| "The tool is built and parked, that's enough" | A reusable tool that is parked and never registered in AGENTS.md is invisible to later sessions. Register the pointer; that is the persistence step. |
