# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-04 — sweep (3 findings)

*Note: findings 1–3 were published in March 2026 and predate the last-checked date of 2026-04-07, but were missed by the initial manual seed. Logged now to close the gap.*

### Finding 1: Orion — first end-to-end ANE training/inference system + 20-constraint catalog
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" bypasses CoreML entirely via private `_ANEClient`/`_ANECompiler` APIs and is the first open system to run both forward and backward passes (109M-param transformer) directly on ANE. Key hardware measurements: ~19 TFLOPS FP16 peak, ~0.095 ms dispatch overhead, 32 MB SRAM performance cliff; ANE utilization reaches 94% at 16–64 op graph depth. Catalogs 20 ANE compiler constraints including 14 previously undocumented (memory layout rules, MIL IR restrictions, per-process compilation limits).
- **Why it matters:** Most thorough public hardware characterization of ANE to date; the constraint catalog and SRAM cliff directly bound what workload shapes keep ANE fully engaged — critical context for interpreting IOReport energy samples as a utilization proxy.

### Finding 2: maderix Part 3 Substack + maderix/ANE open-source repo
- **Source:** maderix Substack / GitHub
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b · https://github.com/maderix/ANE
- **Date:** March 7, 2026
- **Summary:** Part 3 of the maderix M4 ANE series demonstrates full training (forward pass, backward pass, Adam optimizer) of a 109M parameter transformer on ANE via reverse-engineered private APIs. The companion GitHub repo `maderix/ANE` provides the complete implementation, making direct `_ANEClient`-based ANE compute publicly reproducible for the first time; code includes graph compilation, dispatch, and gradient accumulation routines.
- **Why it matters:** `maderix/ANE` is now the canonical open-source reference for the `_ANEClient` API surface; any new telemetry or counter hooks in that private API will surface here first, and the dispatch/timing code is a ready reference for measuring ANE activity without hardware counters.

### Finding 3: lambdafoo mperf — kperf counter topology confirmed (2 fixed + 8 configurable)
- **Source:** lambdafoo.com ("Perpetually Curious Blog")
- **URL:** https://lambdafoo.com/posts/2026-03-25-mperf-hardware-counters-macos.html
- **Date:** March 25, 2026
- **Summary:** Introduces `mperf`, a `perf stat`-style CLI for Apple Silicon using `kperf.framework`/`kperfdata.framework`. Empirically confirms the Apple Silicon PMU topology: **2 fixed counters** (cycles, instructions) + **8 configurable slots** = 10 simultaneous events maximum. Event database lives at `/usr/share/kpep/` as chip-specific plist files; portable aliases (`cycles`, `instructions`, `branch-misses`) resolve to correct event IDs at runtime.
- **Why it matters:** The 8-slot configurable ceiling is the hard budget for t3rm1nu55-monitorplus's kperf sidecar; any counter group design must fit within this limit. The `/usr/share/kpep/` plist structure is directly parseable for dynamic event enumeration.

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
