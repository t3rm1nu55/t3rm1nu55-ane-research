# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-09-05 — sweep (4 findings)

### Finding 1: Comprehensive ANE reverse-engineering paper (arXiv 2606.22283)
- **Source:** arXiv / Spencer H. Bryngelson (Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** 2026-06-21
- **Summary:** "Apple Neural Engine: Architecture, Programming, and Performance" is the most complete public documentation of the ANE to date, covering A11–A18 and M1–M5 SoC families. Based on direct measurement and static analysis of the private runtime, compiler, kernel driver, and firmware, it documents the datapath, dispatch route below CoreML, on-disk program format, weight-compression scheme, and the kernel driver / firmware / command protocol. The companion GitHub repo `sbryngelson/ane-guide` serves as a reference manual.
- **Why it matters:** The kernel driver and command protocol documentation could reveal instrumentation hooks for ANE activity tracking that go beyond IOReport energy deltas.

### Finding 2: ANEForge — Python direct-dispatch library (arXiv 2606.17090)
- **Source:** arXiv / Spencer H. Bryngelson (Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** 2026-06-12
- **Summary:** ANEForge is an open Python package that compiles a lazy tensor graph (58 fused operators, 19 native bridge operators) into an ANE program and dispatches it through the same daemon and kernel-driver stack as Apple's internal framework — bypassing CoreML. It supports LLM decode/prefill, on-ANE training, ONNX import, and vision models. Available on PyPI (`aneforge`) and GitHub (`sbryngelson/ANEForge`).
- **Why it matters:** Dispatching through the native daemon/driver path may expose timing and latency data that could serve as a utilization proxy, and the open dispatch code is the clearest public roadmap to the driver ABI.

### Finding 3: M1 AMX dual-block characterization (arXiv 2606.25426)
- **Source:** arXiv / Deyvik Bhan
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** 2026-06-24
- **Summary:** "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" characterizes the M1 AMX inner loop as load-issue bound (ceiling ~610–680 GFLOPS under load, vs. ~1.4 TFLOPS load-free). The key novel result is the explicit documentation that M1 contains **two on-chip AMX blocks**, and that Accelerate underutilizes the second block — the speedup over Accelerate comes from fine multi-thread panels that fill both blocks.
- **Why it matters:** Confirms two independently schedulable AMX blocks on M1; any future AMX utilization metric in t3rm1nu55-monitorplus should account for per-block scheduling rather than a single aggregate.

### Finding 4: Orion — first open ANE training+inference runtime (arXiv 2603.06728)
- **Source:** arXiv / Ramchand Kumaresan
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** 2026-03-06
- **Summary:** Orion is the first open end-to-end system combining direct ANE execution, a compiler pipeline, and stable multi-step training with checkpoint resume — all bypassing CoreML via `_ANEClient` / `_ANECompiler` private APIs. It catalogs 20 restrictions on MIL IR programs, memory layout, and compilation limits discovered through empirical testing. The GitHub implementation (`mechramc/Orion`) is publicly available.
- **Why it matters:** The 20-restriction constraint catalog implicitly defines the ANE's observable behavioral envelope; anomalous IOReport readings outside the cataloged execution patterns could serve as a utilization heuristic.

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
