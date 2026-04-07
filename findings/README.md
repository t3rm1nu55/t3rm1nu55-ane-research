# Findings

Write-ups of experiments that produced actionable results.

**Status:** empty. Nothing has matured from experimentation to findings yet.

## Format for future findings

When an experiment works, write a markdown file in this directory with the following structure:

```
# <Finding title>

**Date:** YYYY-MM-DD
**Experiment:** link to experiments/<name>
**Journal entry:** link to journal.md#section
**Status:** proposed | validated | integrated

## Background
What problem in the main project this finding addresses.

## Method
How the experiment was set up and run.

## Results
What you observed. Include raw data if small enough.

## Implications
What this means for `t3rm1nu55-monitorplus`. Specifically: which source module
should consume this, what Cargo feature flag it gates behind, and what minimum
macOS / chip version is required.

## Open questions
What we still don't know.
```
