# t3rm1nu55-ane-research

A research journal tracking the **Apple Neural Engine (ANE)** and **AMX matrix coprocessor** counter exposure problem on Apple Silicon.

This is the sister repository to [`t3rm1nu55-monitorplus`](https://github.com/t3rm1nu55/t3rm1nu55-monitorplus), a deep low-level macOS system monitor. That project currently exposes ANE power-gate state (from IOReport's Energy Model channel) but **not** throughput counters — because throughput counters for the ANE and AMX are not publicly documented and have not been fully reverse-engineered.

This repo exists to track the reverse-engineering frontier so that, when the work matures, the findings can land in `t3rm1nu55-monitorplus` as real metrics.

## Scope

- **In scope:**
  - ANE (Apple Neural Engine) performance counter exposure attempts
  - AMX (Apple Matrix Coprocessor) performance counter exposure attempts
  - Related reverse-engineering work: dougallj/applecpu, hollance/neural-engine, MIT CSAIL AMX work, Asahi Linux M-series PMU patches
  - Any arXiv preprints on Apple Silicon deep-counter access
- **Out of scope:**
  - Actively reverse-engineering Apple's IP ourselves (we track others' published work)
  - Anything covered by the public IOReport API (that's `t3rm1nu55-monitorplus`)
  - Non-ANE/AMX work on Apple Silicon (power modeling, thermal, etc.)

## Structure

| File / dir | Purpose |
|---|---|
| `journal.md` | Chronological log of findings. Updated daily by an autonomous research agent. |
| `references.md` | Tracked sources: papers, GitHub repos, blog posts, academic theses. Diff target for the daily agent. |
| `experiments/` | Small Rust probes against private ANE/AMX APIs (when/if something becomes worth testing). |
| `findings/` | Write-ups when an experiment produces something actionable. |

## How the daily agent works

A [RemoteTrigger](https://support.anthropic.com) scheduled job runs a Sonnet agent once per day. The agent:

1. `WebSearch`es the tracked sources for new activity since its last run
2. Diffs findings against `references.md`
3. Appends new discoveries to `journal.md` with date headers
4. Opens an issue on `t3rm1nu55-monitorplus` when something becomes actionable for the main project

No human maintains this repo directly. It is a passive intelligence stream.

## License

MIT. The research notes and journal are MIT-licensed so anyone can freely reference them.
