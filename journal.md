# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-19 — sweep (5 findings)

### Finding 1: Comprehensive ANE reverse-engineering paper — A11 through M5 architecture, kernel driver, command protocol
- **Source:** arXiv 2606.22283
- **URL:** https://arxiv.org/abs/2606.22283 (web edition: https://ane-guide.readthedocs.io)
- **Date:** June 2026
- **Summary:** Spencer Bryngelson (Georgia Tech) published a complete reverse-engineered account of the Apple Neural Engine covering A11–A18 and M1–M5, including the ANE datapath and roofline, dispatch route below CoreML via `_ANEClient`, compiler and on-disk program format, weight-compression scheme, and — critically — the kernel driver, firmware, and command protocol. Direct measurements are on M1 and M5. Companion tools ANEForge and ane-guide are published alongside.
- **Why it matters:** First public documentation of the ANE kernel driver and firmware command protocol; this is the layer where hardware performance counters or telemetry registers would be exposed.

### Finding 2: ANEForge — Python library for direct ANE dispatch without CoreML
- **Source:** arXiv 2606.17090 / GitHub sbryngelson/ANEForge / PyPI aneforge
- **URL:** https://arxiv.org/abs/2606.17090 — https://github.com/sbryngelson/ANEForge
- **Date:** June 12, 2026
- **Summary:** ANEForge compiles a lazy tensor graph (58 fused operators + 19 native bridge operators) and dispatches it to the ANE via private symbols, with no CoreML dependency. Training, forward, backward, and optimizer steps all compile to ANE programs. Available on PyPI as `aneforge`.
- **Why it matters:** Provides an inspectable open-source call path from Python down to `_ANEClient`; tracing IOReport deltas around ANEForge dispatch calls is the most practical current approach to infer ANE utilization in t3rm1nu55-monitorplus.

### Finding 3: Orion — first open end-to-end ANE runtime for LLM training + inference
- **Source:** arXiv 2603.06728 / GitHub mechramc/Orion
- **URL:** https://arxiv.org/abs/2603.06728 — https://github.com/mechramc/Orion
- **Date:** March 6, 2026
- **Summary:** Orion is the first open system combining direct ANE execution, a compiler pipeline, and multi-step training — bypassing CoreML entirely via `_ANEClient` and `_ANECompiler`. The paper extends the public catalog of ANE constraints to 20 restrictions covering MIL IR programs, memory layout, compilation limits, and numerical behavior. It also documents an 8.5× ANE compilation speedup over the CoreML path.
- **Why it matters:** The documented private API surface (`_ANEClient`, `_ANECompiler`) is stable enough to build against; if any internal utilization counter exists, it would be reachable through this call path.

### Finding 4: maderix/ANE — GitHub repo for training directly on ANE via reverse-engineered APIs
- **Source:** GitHub maderix/ANE
- **URL:** https://github.com/maderix/ANE
- **Date:** March 2026 (coincides with maderix "Inside M4 ANE" Substack series)
- **Summary:** maderix followed up their M4 ANE Substack article series with a companion GitHub repo providing working code for training neural networks directly on the ANE using reverse-engineered private APIs (`_ANEClient`/`_ANECompiler`). The Substack article confirmed the "38 TOPS" Apple marketing figure is derived by doubling FP16 performance.
- **Why it matters:** Provides another inspectable implementation of the private ANE dispatch path; cross-referencing with Orion and ANEForge narrows what the private API surface looks like.

### Finding 5: AMX inner-loop bottleneck characterization — first systematic throughput model for M1 AMX
- **Source:** arXiv 2606.25426
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Deyvik Bhan reverse-engineers the M1 AMX inner loop and shows it is load-issue bound: any operand load interleaved with the FMA32 stream drops single-thread throughput to 610–680 GFLOPS, under half the load-free rate. A hand-written AMX kernel beats all Accelerate fp32 GEMM paths at 1.17–1.58× geometric mean for LLM prefill shapes at S=128.
- **Why it matters:** Provides concrete throughput thresholds that could anchor an inference-based AMX utilization model (comparing measured FLOPS against 610 GFLOPS floor vs. ~1.4 TFLOPS ceiling on M1 without dedicated counters).

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
