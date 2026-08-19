---
name: building-deterministic-tools
description: >
  Use when a task must be repeated faithfully — querying a large dataset,
  wrapping an API, consuming a structured source — where a missed detail breaks
  the outcome. Build a small deterministic agent-friendly tool and query it
  instead of re-doing the work by hand. Do NOT activate when the work is small,
  single-use, no outcome depends on a missed detail, or a maintained tool
  already fits.
---

# Building Deterministic Tools

## Overview

An AI agent is probabilistic. Reading 5000 lines of generated markup by eye
and *hoping* nothing is missed is a bet, not a plan. The same miss risk
applies to any repeated faithful work: querying a dataset by hand, wrapping an
API call-by-call, re-reading a source for every answer. Any task whose
correctness depends on missing nothing has a **determinism gap** — close it
with a small deterministic tool that does the work once, then answers the
agent's questions about it one at a time.

**The spine: determinism makes you repeatable; reachable-and-verified make you
right.**

- **Deterministic** — same input, same output, every run.
- **Reachable (or loud)** — every answer the task can need can be pulled
  through a command, or is flagged loudly as unresolved.
- **Verified** — output is checked by a route other than the one that produced
  it, before trust.

Determinism without reachable-and-verified replaces probabilistic misses with
deterministic lies: confident, repeatable, wrong. The extractor that ran clean,
produced a tidy index, and silently dropped data. Or the friendly-looking tool
that cannot reach the one element the task needs.

The reader of the output is also probabilistic: the answers feed a later
session whose belief is a next-token prediction. Evidence must be re-derivable
from the ground truth, not believed.

## When NOT to Use

- The work is small enough to do fully by hand, and done once.
- A maintained tool for the work already exists.
- The output is consumed only by this session and never verified against the
  ground truth.
- No outcome depends on missing any detail.

---

## Phase 1: Decide — build the tool or do the work by hand

**Entry:** The task involves repeated faithful work over something with ground
truth.

**Exit:** A verdict: `build` or `eyeball`.

**Steps:**

1. Measure the work: how many items, rows, calls, or elements repeat? Large
   means hundreds or thousands of repetitive, machine-generated units.
2. Assess the stakes: does a missed detail break the outcome? Listen for the
   user's own words — "don't miss any detail", "every X must be preserved".
3. Assess reuse: will the answers feed later sessions, a redesign, or a
   verification against the ground truth?
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

- **Reusable** — the work recurs across sessions or sources: the same kind of
  query, the same kind of source shape, the same kind of wrapping. Choose this
  by default; the build cost is paid once and reused.
- **Throwaway** — the work is provably one-off: a specific input that will
  never recur. Use the lean park-in-scratch path.
- Choose `reusable` unless the one-off case is established. A reusable tool's
  cost is paid once and reused; a throwaway tool's cost is paid every time the
  work recurs.

**Failure behavior:** when the work's recurrence is unclear, choose `reusable`
— the persistence step is cheap, and re-doing recurring work is not.

---

## Phase 3: Build the tool — deterministic and agent-friendly

**Entry:** Verdict is `build`; lifecycle is chosen.

**Exit:** The tool does the work once (parses the source, indexes the dataset,
wraps the API), exposes the fixed documentation core (overview, catalog, help),
and answers each question through a retrieval path that returns a complete,
untruncated result. Every answer the task can need is reachable from the
overview. Re-running produces byte-identical output.

**Steps — the deterministic core:**

1. Do the work with a real mechanism — a real parser (`html.parser.HTMLParser`,
   `json`, `csv`), a real query (SQL, an index), a real API client. Hand-rolled
   string slicing is a miss risk, not a shortcut.
2. Record **every** attribute, key, and field the work touches; buffer raw
   embedded content verbatim.
3. Resolve identifiers by **name**, not by value. A token reference like
   `var(--ink-ink, ...)` matches its app counterpart on the token NAME because
   the two sides store values in different spaces (app `oklch()` vs design hex
   fallback). Matching on the hex value fails silently.
4. The raw dump is complete — render every key, filter nothing. Filtering is
   where data dies.
5. Report unresolved items as a separate, explicit list (unmatched vars, raw
   colors, unanswered queries). They are decisions for the consumer, not noise.
6. Name output per source (`<slug>-dump.json`, ...) so multiple inputs never
   collide.
7. Derive everything downstream — the overview and every answer — **from the
   dump or index**, never by re-doing the work. A bug fix then re-derives in
   seconds and a re-run never touches the ground truth a second time.

**Steps — the fixed documentation core** (every tool has all three; this shape
is what any agent can rely on):

- `<tool> overview` — one-glance digest: counts, distinct keys, queryable
  names, + verification receipt. The agent reads this first; it must fit in one
  glance.
- `<tool> catalog` — self-description: list of subcommands, each with
  description, param schema (types/valid values/examples), and what it returns.
  New agents learn the whole surface from the tool itself.
- `<tool> help <subcommand>` — per-command detail.

**Steps — scenario-built retrieval commands** (free-form, built per scenario,
one command per element type / query pattern). Each follows MCP-inspired
discipline, adapted for agents:

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
walk from the overview to any answer by composing its commands.

**Failure behavior:** if the work throws mid-way, print the offending input to
stderr and skip that item — never emit a partial result as if it were complete.

---

## Phase 4: Verify, then persist

Verify by a route other than the one that produced the answer. The code path
that emitted the output must never be the path that certifies it — a
self-derived count only proves the tool agrees with itself.

**Entry:** The tool answers through its interface.

**Exit:** The scenario's verification patterns pass; the tool is persisted
along the Phase 2 path; the actual task resumes against the tool.

**The verification meta-rule:** name the ground truth the work is faithful to,
then verify by a route the tool did not produce. The ground truth is the
source file, the dataset, the API's responses, or the corpus — whatever the
tool's correctness is measured against.

**Per-scenario verification patterns live in `verification-patterns.md`** —
pick the row for the scenario's ground truth and run it. The patterns include
the four source checks (re-run diff, independent count, spot-checks, reachability
sweep) plus dataset, API, and corpus rows. For a scenario with no row, apply
the meta-rule directly: choose checks the tool did not produce, and make them
checkable.

1. Run the scenario's verification patterns.
2. **Documentation core — catches invisibility.** Confirm the tool answers
   `overview`, `catalog`, and `help <subcommand>`; a tool that cannot describe
   itself is invisible to later sessions.
3. **Persistence — catches orphaned tools.** On the reusable path, register the
   tool in the repo's `AGENTS.md` — add one entry under `## Built tools`:

   ```markdown
   - `<tool>` — `<location>` — invoke with `<command>` — answers `<what>` —
     query when `<trigger description>`
   ```

   The **trigger description** tells later sessions when to query this tool
   instead of re-doing the work. The tool stays out of the product, the build,
   and CI. The checks are the test — no test files, no committed fixtures, no
   generalization, no config surface. A fixture is just input the tool already
   handled, so it cannot catch what the tool cannot see.
4. Only when every item passes, append the receipt to the overview: `verified:
   idempotent, count = N, spot-check = 3/3`. Then consume the tool.

**Persist along the Phase 2 path:**

- **Reusable:** persist under `skills/building-deterministic-tools/tools/<slug>/`
  (tool + dump + catalog). Register the pointer in `AGENTS.md` as above.
- **Throwaway:** park it in the scratch dir, keep it lean, discard after the
  task completes.

**Consume:** query the tool for every answer. Re-doing the work by hand
re-introduces the miss risk the tool removed; evidence comes from the tool, and
the tool is verified.

**Failure behavior:** never consume unverified output. Any red check means the
tool is wrong, and the task must wait for the fix.

---

## Anti-Rationalization Table

| Rationalization | Redirect |
|---|---|
| "This is small, I can do it by hand" | Small is fine to eyeball — but measure first. The moment it is large, high-stakes, or reused, build the tool. |
| "I know the output is right, I saw it" | **Hard stop.** Seeing is probabilistic; a re-run against the known-good count is deterministic. Run the gate. |
| "The tool's count proved the output is complete" | **Hard stop.** Self-reported verification. The count only proves the tool agrees with itself — re-derive from the ground truth with an independent count. |
| "I'll just re-do the work when I need a value" | Re-doing per value is slow and re-introduces the miss risk. The overview names it; one retrieval call returns it whole. |
| "The values didn't line up, I'll match by the raw value" | The two sides store values in different spaces (oklch() vs hex). Resolve by name; report unmatched as decisions. |
| "This tool is useful, let me wire it into the repo properly" | The skill is about the *work* being deterministic, not the tool being permanent. Park it; resume the task. |
| "I dropped those fields — they weren't important" | **Hard stop.** Reachable or loud. If it isn't in the dump, it is lost. Render everything or flag it. |
| "The counts matched, so nothing was dropped" | Counts cannot see per-item loss. Spot-check a retrieval path; reachable or loud. |
| "One command that prints everything is simpler" | A single fat call fills the context and buries the detail the task needs. Slice it into an overview plus targeted paths. |
| "That item class doesn't need a path, the task won't ask" | Reachability is the contract. Every class the overview names must have a working path; the consumer decides what it asks for. |
| "I'll skip the catalog, the overview says enough" | **Hard stop.** A tool without a catalog is invisible: new agents can't learn its surface. The catalog is the self-description; the overview is the digest. |
| "That item class isn't reachable, but I can read it by hand" | **Hard stop.** Reachable or loud is the contract. If a class has no path, the tool is wrong — fix the tool, and the task waits. |
| "This work might recur, but the throwaway path is faster" | Choose reusable unless the one-off case is established. A reusable tool's cost is paid once; re-doing recurring work is paid every time. |
| "The tool is built and parked, that's enough" | A reusable tool that is parked and never registered in AGENTS.md is invisible to later sessions. Register the pointer; that is the persistence step. |
| "This scenario has no verification row — I'll skip verifying" | **Hard stop.** The meta-rule always applies: name the ground truth, verify by a route the tool did not produce. No row means apply the meta-rule, not skip it. |
