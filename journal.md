# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-04-09 — sweep (3 findings)

### Finding 1: Orion — First open LLM training/inference system on ANE via private APIs
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026
- **Summary:** Ramchand Kumaresan's Orion bypasses CoreML entirely by calling `_ANEClient` and `_ANECompiler` private APIs directly, delivering the first open system that both trains and runs LLMs on the ANE. It catalogs 20 ANE constraints (14 previously undocumented), demonstrates 94% ANE utilization achievable via deep operation graphs (16–64 ops), and documents an ANE compiler limit of ~119 compilations per process before silent failures begin. Companion code at https://github.com/mechramc/Orion.
- **Why it matters:** The most detailed public catalogue of `_ANEClient`/`_ANECompiler` API surface to date; the utilization measurement methodology and constraint catalogue are essential groundwork before t3rm1nu55-monitorplus can attempt any ANE instrumentation.

### Finding 2: maderix Part 3 — ANE training, weight-blob crack, first M5 data point
- **Source:** maderix Substack
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** ~March 2026 (per companion repo commit history)
- **Summary:** Part 3 of the maderix M4 ANE series covers training neural networks on ANE: reverse-engineering the weight blob format, implementing a delta-compilation workaround for the 119-compile limit, and reporting the first public M5 ANE throughput data point (via contributor m0at). Companion code at https://github.com/maderix/ANE gained INT8 W8A8 quantization (March 10, 2026), achieving 1.88× throughput improvement on M4.
- **Why it matters:** The M5 data point and INT8 throughput characterisation extend the known ANE performance matrix; the companion repo is an active working reference for direct ANE access patterns that no other tracked project currently provides.

### Finding 3: ClF3 blog — PMU control register layout differs on M3/M4 vs M1/M2
- **Source:** ClF3's blog
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** Unknown (post-January 2026, untracked source)
- **Summary:** Documents that `SYS_APL_PMCR0_EL1` and `SYS_APL_PMCR1_EL1` event-selector fields are 16 bits per slot on M3/M4 but only 8 bits on M1/M2, a breaking difference not noted in the bugsiki blog (which was M2-based). This affects how kperf counter configuration code must be written to support M3/M4.
- **Why it matters:** The kperf FFI in t3rm1nu55-monitorplus needs chip-generation-aware PMU register layout; this is the first public documentation of exactly where M1/M2 and M3/M4 bitfield widths diverge.

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
