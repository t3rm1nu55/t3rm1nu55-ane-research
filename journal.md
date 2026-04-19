# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-04-19 — sweep (2 findings)

*Note: both findings were published in March 2026 but were missed by the manual initial seed on 2026-04-07. This is the first automated sweep run; items not in the journal are logged regardless of publication date.*

### Finding 1: Orion — first open end-to-end ANE runtime with utilization measurement
- **Source:** arXiv 2603.06728 + GitHub mechramc/Orion
- **URL:** https://arxiv.org/abs/2603.06728 / https://github.com/mechramc/Orion
- **Date:** March 2026
- **Summary:** Orion bypasses CoreML entirely via Apple's private `_ANEClient` and `_ANECompiler` APIs, delivering 170+ tok/s inference and full transformer training on the ANE. The paper catalogs 20 ANE constraints (14 previously undocumented), including the ~119-compile-per-process limit that Orion circumvents with delta compilation. The benchmark suite measures ANE utilization as fraction of wall time spent inside `orion_eval` dispatches — the closest thing to a real utilization metric yet published.
- **Why it matters:** The `orion_eval` timing ratio is an adaptable methodology for ANE utilization estimation in t3rm1nu55-monitorplus without needing hardware counters.

### Finding 2: maderix Part 3 — transformer training on ANE completes three-part series
- **Source:** maderix Substack
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** 2026-03-07
- **Summary:** The third and final installment of maderix's ANE reverse-engineering series demonstrates full backward-pass training (forward, backward, gradients, Adam optimizer) on the M4 ANE via `_ANEClient`, including Qwen3-0.6B (596M parameters). The initial seed only referenced Part 2 (benchmarks). Part 3 confirms the `_ANEClient` private API is stable enough for gradient workloads, not just inference.
- **Why it matters:** Confirms `_ANEClient` dispatch stability under sustained load — relevant for any telemetry sidecar that hooks into ANE dispatch events.

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
