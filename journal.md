# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-04-13 — sweep (3 findings)

> **Sweep note:** No new activity was found in any tracked source during the April 7–13 window (all monitored GitHub repos returned zero commits; no new LKML Apple PMU patches; no confirmed post-April-7 papers or blog posts). The three findings below are substantive items published in March 2026 that were not captured in the April 7 initial seed. They are logged here to complete the baseline.

### Finding 1: maderix ANE series Part 3 + open-source ANE runtime (maderix/ANE)
- **Source:** maderix Substack (Part 3) / GitHub maderix/ANE
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b / https://github.com/maderix/ANE
- **Date:** March 7, 2026
- **Summary:** Part 3 of the maderix M4 ANE series demonstrates full transformer training on the ANE (forward pass, backward pass, gradient computation, Adam optimizer), scaling to Qwen3-0.6B with 596M parameters. The companion GitHub repo releases the complete Objective-C runtime under MIT. The reported "11.2% ANE utilization" is measured as the fraction of wall-clock time the ANE is actively executing, not via hardware counters.
- **Why it matters:** Canonical open-source reference for `_ANEClient`/`_ANECompiler` direct-access patterns that bypass CoreML; sets the current ceiling for ANE observability without hardware counters and is the code base any counter-based approach would be validated against.

### Finding 2: "Orion" — full ANE LLM runtime (arXiv 2603.06728 + mechramc/Orion)
- **Source:** arXiv / GitHub mechramc/Orion
- **URL:** https://arxiv.org/abs/2603.06728 / https://github.com/mechramc/Orion
- **Date:** March 2026
- **Summary:** Academic paper and open-source implementation of a compiler + runtime for LLM training and inference that runs directly on the ANE, bypassing CoreML and Metal. Catalogues 20 ANE compiler restrictions (14 newly discovered MIL IR, memory, and I/O constraints) and reports 94% ANE utilization for deep op graphs (16–64 ops) — measured via benchmark throughput, not hardware counters.
- **Why it matters:** Most complete public characterization of ANE operational constraints to date; the 20-restriction catalog is a new reference for understanding what the ANE compiler rejects, and the throughput measurements are the best available proxy for ANE utilization until hardware counters exist.

### Finding 3: clf3.org — M3/M4 PMU ESR register layout differs from M1/M2
- **Source:** clf3's blog
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** Unknown (surfaced in searches; direct access returned 403)
- **Summary:** Documents that M3 and M4 PMU ESR registers use 64-bit fields with 16-bit per-event encoding, distinct from the layout used on M1 (confirmed independently by LKML's apple_m1 PMU driver patches, which note per-implementation differences that required the v9 patchset's `per-implementation PMU startup` support). Source is search-snippet only — primary verification blocked.
- **Why it matters:** The kperf sidecar in t3rm1nu55-monitorplus configures PMU event fields; any code that assumes M1/M2 event-field widths will silently misconfigure counters on M3/M4. This needs explicit chip-generation detection before counter reads can be trusted across the full M-series range.

---

## 2026-04-07 — Initial seed

Repository created. Initial scope, structure, and references.md seeded from a research synthesis produced on 2026-04-06 by a Sonnet agent investigating the open problems in Apple Silicon deep telemetry.

**Seed findings (the starting state of the field):**

### Apple Neural Engine (ANE)

- The ANE has **no public performance counter API**. Throughput is not directly measurable.
- ANE **power** *is* visible through IOReport's "Energy Model" channel on M-series chips — this gives millijoule-resolution energy consumption per sample interval.
- On/off **gate state** is detectable: when the ANE is not in use, it is hard-power-gated and consumes zero energy.
- Direct ANE benchmark access has been demonstrated via private `_ANEClient` symbols resolved at runtime (see [hollance/neural-engine](https://github.com/hollance/neural-engine)), bypassing CoreML's abstraction layer.
- The [maderix ANE M4 analysis](https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-615) measured M4 ANE peak at 19 TFLOPS FP16 at 2.8W — this is the current best public characterization of ANE throughput and power, derived via black-box benchmarking rather than counters.

### AMX Matrix Coprocessor

- AMX is **undocumented** by Apple. It is a matrix coprocessor integrated into each P-core that accelerates matrix multiplication; used by Accelerate framework's BLAS implementations.
- The [MIT CSAIL AMX SB thesis (Jonathan Zhou, 2025)](https://commit.csail.mit.edu/papers/2025/Jonathan_Zhou_SB_Thesis.pdf) is the deepest public treatment of AMX performance but measures at algorithmic granularity, not via hardware counters.
- [dougallj/applecpu](https://github.com/dougallj/applecpu) documents the Firestorm/Icestorm microarchitecture and some AMX instruction encoding.
- kperf/kpc exposes 8 programmable counters on M1–M4 with ~60 named events, but there are **no documented AMX-specific events** in the counter lists that have been extracted.

### The open problem

No tool — open source or proprietary — exposes real-time ANE or AMX utilization as a hardware-counter-derived metric. The best you can do today is:
- **Power indirection:** sample ANE energy deltas over known intervals from IOReport Energy Model, and infer "ANE active" when power > 0.
- **Dispatch pattern inference:** for AMX, detect AMX instruction sequences in user code via Mach-O disassembly (static) or via CoreFoundation/Accelerate API hooking (dynamic, fragile).
- **Benchmark proxies:** run a known workload and measure wall-clock time.

None of these give you the "ANE is currently at 42% utilization" metric that users would actually want.

### What would "solving this" look like

1. **ANE hardware counter reverse engineering:** find an equivalent of kperf for the ANE's internal performance counters (if they exist and are accessible from the host CPU).
2. **AMX event discovery:** scan the kperf event database across chip generations for AMX-labeled events (they may be hidden under generic "vector unit" naming).
3. **Private framework hooks:** discover a private CoreML/ANECompiler symbol that exposes utilization metrics to Apple's internal profiling tools.

This repository will track any of these approaches as they emerge upstream.
