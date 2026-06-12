# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-12 — sweep (3 findings)

### Finding 1: clf3.org — M3/M4 PMU register format change documented

- **Source:** ClF3's blog
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** Q1 2026 (exact date not available in search results)
- **Summary:** Documents a breaking change in the M3/M4 PMU register layout: ESR registers are now 64-bit (up from 32-bit), and each event selector takes 16 bits instead of 8 bits, meaning `SYS_APL_PMCR0_EL1` and related registers must be written differently on M3+ vs M1/M2. The post provides concrete register definitions and bit-field layouts for both generations.
- **Why it matters:** The kperf sidecar in t3rm1nu55-monitorplus likely encodes PMU event selectors using M1/M2 field widths; this must be handled per-chip-generation or counters will silently program the wrong events on M3/M4 hardware.

### Finding 2: maderix/ANE open-source repo + Substack Part 3 (training on ANE)

- **Source:** maderix Substack + GitHub
- **URL:** https://github.com/maderix/ANE · https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** March 7, 2026 (Part 3 publication); repo released early March 2026
- **Summary:** maderix released a MIT-licensed open-source Rust/ObjC implementation of a full transformer training loop running directly on the ANE via reverse-engineered private APIs (`_ANEClient`, `_ANECompiler`, `_ANEInMemoryModelDescriptor`). The repo achieves 9.3 ms/step for a 768-dim transformer layer on M4, with utilization measured by throughput ratio against theoretical peak (not hardware counters). Part 3 scales this to Qwen3-0.6B (596M parameters) with 72 ANE kernels per compile.
- **Why it matters:** First open-source code demonstrating the complete `_ANEClient`/`_ANECompiler` private API call sequence for custom graph execution; the codebase is a directly readable reference for any future ANE dispatch-rate telemetry work in monitorplus.

### Finding 3: Orion paper — most comprehensive public ANE characterization to date

- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 2026
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" builds on the maderix foundation and extends the public ANE constraint catalog from 6 to 20 rules (14 newly discovered), including MIL IR, memory, and I/O constraints. The system uses IOSurface-backed zero-copy tensor I/O and a 5-pass compiler pipeline; on M4 Max it achieves 170+ tokens/s for GPT-2 124M inference and trains a 110M-parameter transformer in 22 minutes. The ~119 compile-per-process limit (requiring process restart to continue) is confirmed as a hard ceiling.
- **Why it matters:** The IOSurface zero-copy pattern and the 119-compilation ceiling are the two implementation constraints any future ANE integration in monitorplus must work around; the 20-constraint catalog is now the authoritative reference for what ANE graph shapes are valid.

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
