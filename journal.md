# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-14 — sweep (5 findings)

### Finding 1: Apple Neural Engine: Architecture, Programming, and Performance (arXiv 2606.22283)
- **Source:** arXiv — Spencer H. Bryngelson (Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.22283 — web edition: https://ane-guide.readthedocs.io — code: https://github.com/sbryngelson/ane-guide
- **Date:** June 21, 2026
- **Summary:** Comprehensive reverse-engineered treatment of the ANE covering A11–A18 and M1–M5 chip families, derived from direct measurement and static analysis of private Apple frameworks. Documents the datapath, roofline throughput/energy bounds, dispatch route, MIL compiler and program format, weight-compression scheme, and—critically—the kernel driver command protocol and firmware. Per-chip target tables and an operation-by-device matrix are included.
- **Why it matters:** The kernel driver command protocol documentation is the most direct public resource for identifying whether any telemetry or timing hooks exist in the ANE dispatch path; could reveal a non-counter utilization signal at the driver layer.

### Finding 2: ANEForge — Python for direct computation on the Apple Neural Engine (arXiv 2606.17090)
- **Source:** arXiv / PyPI / GitHub (sbryngelson/ANEForge)
- **URL:** https://arxiv.org/abs/2606.17090 — code: https://github.com/sbryngelson/ANEForge
- **Date:** June 12, 2026
- **Summary:** Open-source Python package that compiles a lazy tensor graph (58 fused + 19 native bridge operators) into ANE programs and dispatches them through the ANE daemon and kernel-driver stack without any CoreML dependency. Forward pass, backward pass, and Adam optimizer updates all run as ANE programs. Available on PyPI.
- **Why it matters:** Provides a programmable ANE workbench; controlled dispatches via ANEForge paired with IOReport energy-delta sampling could calibrate and validate the power-indirection utilization approach in t3rm1nu55-monitorplus.

### Finding 3: Above the Inner Loop — AMX microarchitecture on M1 (arXiv 2606.25426)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Characterizes the M1 AMX inner loop as load-issue bound, with single-thread throughput collapsing from theoretical peak to a 610–680 GFLOPS band whenever an operand load interleaves with the FMA32 stream. Discovers that M1 hosts two on-chip AMX blocks exploitable via fine multi-thread panels; beats Accelerate's fastest FP32 path 1.17x on LLM-scale prefill GEMMs.
- **Why it matters:** Two-AMX-block topology and load-issue bound are new public AMX microarchitecture facts; provides context for interpreting kperf L1/L2-miss and vector-unit counter patterns during AMX-active workloads.

### Finding 4: Orion — Direct ANE runtime for LLM training and inference (arXiv 2603.06728)
- **Source:** arXiv / GitHub (mechramc/Orion)
- **URL:** https://arxiv.org/abs/2603.06728 — code: https://github.com/mechramc/Orion
- **Date:** March 6, 2026 (pre-sweep-window; uncaptured in seed)
- **Summary:** End-to-end ANE runtime bypassing CoreML entirely via `_ANEClient` and `_ANECompiler`, implementing IOSurface-backed zero-copy tensor I/O, program caching, and delta compilation. Reduces recompile latency 8.5× (4,200 ms → 494 ms per step); achieves 170+ tokens/s for GPT-2 124M on M4 Max.
- **Why it matters:** Provides the deepest public documentation of `_ANEClient`/`_ANECompiler` private API surface and IOSurface dispatch mechanics—the symbols any ANE dispatch-interception approach would need to hook.

### Finding 5: maderix — Inside the M4 ANE Part 3: Training + companion open-source repo
- **Source:** maderix Substack / GitHub (maderix/ANE)
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b — code: https://github.com/maderix/ANE
- **Date:** March 2026 (pre-sweep-window; uncaptured in seed)
- **Summary:** Part 3 implements a full transformer forward+backward pass on the M4 ANE via reverse-engineered `_ANEClient`/`_ANECompiler`, achieving 1.78 TFLOPS sustained and 11.2% ANE utilization at 9.3 ms/step; companion GitHub repo is public. The 11.2% utilization figure is derived from IOReport Energy Model power deltas.
- **Why it matters:** Confirms IOReport Energy Model power-indirection as the de-facto community standard for ANE utilization estimation; the open-source implementation is a direct reference for t3rm1nu55-monitorplus's own IOReport-based ANE metric.

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
