# Building Deterministic Tools

An agent skill for turning repetitive, detail-sensitive work into a small tool
that produces reliable, queryable evidence.

## What is this skill?

AI agents are useful at reasoning, but they can miss details when they
repeatedly inspect large or structured inputs by hand. This skill helps an
agent recognize that risk and build a focused tool instead: parse the source
once, preserve the complete result, expose targeted queries, and verify the
answers against the original ground truth.

The goal is not to build a large framework. It is to close the gap between
"the agent probably saw everything" and "every answer can be reproduced and
checked." The detailed workflow and behavior contract are in
[`SKILL.md`](skills/building-deterministic-tools/SKILL.md).

## Why is it helpful?

- **Fewer missed details:** large datasets, generated markup, APIs, and
  document collections are handled by repeatable code rather than visual
  scanning.
- **Reproducible answers:** the same input produces the same output, making
  results easier to compare, debug, and reuse in later sessions.
- **Complete retrieval:** an overview and targeted commands make each relevant
  item reachable without flooding the agent's context.
- **Explicit uncertainty:** unresolved or unmatched data is reported instead
  of being silently discarded.
- **Independent verification:** counts, spot checks, and reachability checks
  test the tool against the source rather than trusting its own summary.

## When should you use it?

Use the skill when work is repeated, large, high-stakes, or likely to be reused
and correctness depends on not missing an item. Typical examples include
extracting every property from generated HTML, querying a large dataset,
wrapping an API for repeated questions, or indexing a document corpus.

For a small one-off task, or when a well-maintained tool already provides the
needed answers, working directly is usually simpler. The skill is a guide for
closing a determinism gap, not a requirement to build a tool for every task.

## What does it provide?

The skill guides the agent toward a compact tool with:

- a one-glance overview of what was found and how it was verified;
- a catalog and command-level help so later agents can discover its interface;
- focused retrieval commands that return complete, bounded results; and
- verification patterns for structured sources, datasets, APIs, and corpora.

For the scenario-specific verification checks, see
[`verification-patterns.md`](skills/building-deterministic-tools/verification-patterns.md).

## Installation

```bash
npx skills add B-R-P/building-deterministic-tools
```

Once installed, invoke the `building-deterministic-tools` skill when a task
requires faithful, repeatable work over a source with ground truth.

