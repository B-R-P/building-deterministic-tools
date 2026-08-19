# Verification Patterns

Per-scenario verification for tools built by `building-deterministic-tools`. Pick
the row whose ground truth matches the scenario, and run its patterns. For a
scenario with no row, apply the meta-rule from SKILL.md: name the ground truth,
then verify by a route the tool did not produce.

Every pattern shares the same purpose — catch the failure a self-derived check
cannot. The code path that emitted the output must never be the path that
certifies it.

## Structured source

**Ground truth:** the source file.

- **Re-run, diff — catches nondeterminism.** Run the tool unchanged and diff
  the raw output against the previous run. Any difference that no source
  change explains means the tool is nondeterministic (set/dict ordering,
  timestamps). Fix and re-run.
- **Independent count — catches quantity loss.** Count elements against the
  *source itself* by a second method (e.g. `grep -c` on the raw file), not
  from the tool's reported count. A mismatch means the extractor dropped
  whole items. This is the check that catches a flush-buffer bug: re-running
  the tool cannot, because a deterministic bug reproduces every run.
- **Spot-check through a retrieval path — catches quality loss.** Pick 2–3
  known elements from the source and pull each *through the interface command
  that claims to reach it*. Confirm the answer is complete: correct resolved
  token, intact text, clean encoding. Counts cannot see per-element key loss —
  this is the load-bearing check, because a layer count can match while every
  layer silently lost its `outline` and `border-left`.
- **Reachability sweep — catches dead ends.** The overview names element
  classes; the independent count sizes them. For each class, confirm a
  retrieval path exists and return one sampled element complete. If a class
  has no path, the tool cannot be trusted to answer the task.

## Dataset / database

**Ground truth:** the dataset rows or the database.

- **Independent count — catches quantity loss.** Count rows by a second method
  (a separate query, a `SELECT COUNT(*)` the tool never runs) and compare with
  the tool's reported count.
- **Spot-check through a retrieval path — catches quality loss.** Pick 2–3
  known records and pull each through the interface command that claims to
  reach it; confirm the answer is complete and matches the record.
- **Reachability sweep — catches dead ends.** Every field the overview names
  must have a working query path.
- **Re-run, diff — catches nondeterminism.** Re-run against a stable snapshot
  and diff; any change no data change explains means the tool is
  nondeterministic.

## Live API

**Ground truth:** the API's responses.

- **Snapshot, re-run, diff — catches nondeterminism.** Capture responses once;
  re-run against the snapshot and diff. The snapshot is the independent route —
  the tool's own fetch path never certifies itself.
- **Spot-check through a retrieval path — catches quality loss.** Pick 2–3
  known responses and pull each through the interface command that claims to
  reach it; confirm the answer is complete.
- **Reachability sweep — catches dead ends.** Every endpoint the overview names
  must have a working retrieval path.
- **Independent count — catches quantity loss.** Count responses by a second
  method (a separate call, a header the tool never reads) and compare with the
  tool's reported count.

## Document corpus

**Ground truth:** the corpus.

- **Independent count — catches quantity loss.** Count documents by a second
  method (filesystem count, a separate index) and compare with the tool's
  reported count.
- **Spot-check through a retrieval path — catches quality loss.** Pick 2–3
  known documents and pull each through the interface command that claims to
  reach it; confirm the answer is complete.
- **Reachability sweep — catches dead ends.** Every document class the overview
  names must have a working retrieval path.
- **Re-run, diff — catches nondeterminism.** Re-run against a stable snapshot
  and diff; any change no corpus change explains means the tool is
  nondeterministic.

## Anything else

**Ground truth:** whatever the tool's correctness is measured against — name it
explicitly before verifying.

Apply the meta-rule: verify by a route the tool did not produce, and make each
check observable. Two checks that work for most ground truths:

- **Independent count** — count the items by a second method and compare with
  the tool's reported count.
- **Spot-check through a retrieval path** — pull 2–3 known items through the
  interface command that claims to reach them; confirm each answer is complete.

If neither fits, the check is still required — name the ground truth, then
choose the independent route that certifies the tool's answers.
