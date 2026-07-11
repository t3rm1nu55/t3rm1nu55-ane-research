# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-11 — sweep (3 findings)

### Finding 1: "Apple Neural Engine: Architecture, Programming, and Performance" (arXiv 2606.22283)
- **Source:** arXiv (Spencer H. Bryngelson)
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 21, 2026
- **Summary:** Reverse-engineered account of the ANE derived from direct measurement on Apple silicon and static analysis of the private runtime, compiler, kernel driver, and firmware. Documents the datapath and throughput roofline, the dispatch route below Core ML, the on-disk compiled program format and weight-compression scheme, and the kernel driver command protocol with firmware.
- **Why it matters:** Most complete public ANE architecture reference ever published; the kernel driver and firmware command-protocol sections are the closest thing yet to a roadmap for instrumenting ANE activity in t3rm1nu55-monitorplus.

### Finding 2: "ANEForge: Python for direct computation on the Apple Neural Engine" (arXiv 2606.17090)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 2026
- **Summary:** ANEForge is an open Python package that dispatches arbitrary compute graphs directly to the ANE using the reverse-engineered private runtime APIs documented in 2606.22283, with no CoreML layer. It provides a working implementation of the full ANE command-dispatch path.
- **Why it matters:** ANEForge's IOKit/kernel call sequence is a live reference for which system interfaces to hook when detecting ANE activity; its dispatch latency measurements could inform the power-indirection threshold used in t3rm1nu55-monitorplus.

### Finding 3: "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" (arXiv 2606.25426)
- **Source:** arXiv (Deyvik Bhan)
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** First public paper to directly program Apple Silicon AMX for LLM GEMM workloads without Accelerate. Shows that gains come from multi-thread panel sizing that saturates M1's two on-chip AMX blocks and from pre-packed weight buffers, beating the fastest Accelerate fp32 GEMM path by 1.17×.
- **Why it matters:** Establishes that M1 has two AMX blocks and characterizes their saturation threshold — directly actionable for AMX utilization inference in t3rm1nu55-monitorplus without requiring undiscovered hardware counters.

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
