# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-12 — sweep (4 findings)

### Finding 1: Orion — first open ANE characterization paper with 20-constraint catalog
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" (Ramchand Kumaresan) is the deepest public academic treatment of the ANE to date. It documents 20 constraints on MIL IR programs, memory layout, and I/O — 14 of which were previously undiscovered — and demonstrates stable multi-step LLM training directly on ANE by bypassing CoreML entirely via `_ANEClient`/`_ANECompiler`. The companion repo is at https://github.com/mechramc/Orion. A key instrumented finding: deep op graphs (16–64 ops) achieve ~94% ANE utilization, and each process is hard-limited to ~119 ANE compilations before subsequent compilations silently fail.
- **Why it matters:** The 20-constraint catalog and the 119-compilation-limit discovery are the most actionable public intelligence yet on ANE internals; the utilization measurement methodology (wall-clock throughput under known graphs) is the best available proxy for ANE activity without dedicated hardware counters.

### Finding 2: maderix Part 3 + maderix/ANE repo — open-source transformer training on ANE
- **Source:** maderix Substack / GitHub
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b  |  https://github.com/maderix/ANE
- **Date:** March 7–10, 2026
- **Summary:** Part 3 of the maderix ANE series demonstrates a full forward+backward pass with Adam optimizer on a 109M-parameter transformer running on the ANE — hardware Apple has never publicly supported for training. The companion GitHub repo `maderix/ANE` publishes all reverse-engineered `_ANEClient`/`_ANECompiler` access code, weight-blob format cracking, and the workaround for the 119-compile limit. This was not captured in the initial April 7 seed (which only tracked Part 2).
- **Why it matters:** First open-source code that successfully drives the ANE's private API for non-inference workloads; the techniques for routing around compilation limits and inspecting compiled graph state are directly relevant to any tool trying to infer ANE utilization from dispatch patterns.

### Finding 3: mperf — new perf-stat-like CLI wrapping kperf on Apple Silicon
- **Source:** Perpetually Curious Blog (lambdafoo.com)
- **URL:** https://lambdafoo.com/posts/2026-03-25-mperf-hardware-counters-macos.html
- **Date:** March 25, 2026
- **Summary:** `mperf` is a new open-source `perf stat`-like CLI for Apple Silicon that wraps the private `kperf.framework`/`kperfdata.framework` APIs. It provides portable event aliases (cycles, instructions, branch-misses, l1d-cache-misses, etc.) that resolve to the correct chip-specific event names from the `/usr/share/kpep/` plist database, outputs JSON, and is scriptable. Exposes the standard 2 fixed + 8 configurable counter slots.
- **Why it matters:** A new minimal reference implementation of kperf counter access on Apple Silicon; its portable alias layer and kpep plist resolution logic are directly relevant to the kperf FFI design in t3rm1nu55-monitorplus.

### Finding 4: NPUMoE — first systematic measurement of ANE dispatch overhead
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2604.18788
- **Date:** April 20, 2026
- **Summary:** "Efficient Mixture-of-Experts LLM Inference with Apple Silicon NPUs" (Benazir & Lin) presents NPUMoE, a runtime that profiles ANE dispatch overhead via offline calibration — measuring expert capacity and popularity to drive static tiling and load-aware graph residency decisions. Results show 1.32–5.55× latency reduction and 1.81–7.37× energy efficiency improvement versus CPU/GPU baselines on M-series hardware.
- **Why it matters:** The offline calibration methodology (timing ANE launches under varying workloads) is a practical blueprint for indirect ANE utilization inference; no new counters are exposed but the quantified dispatch-overhead model is the most rigorous public data on ANE scheduling latency to date.

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
