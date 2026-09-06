# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-09-06 — sweep (5 findings)

### Finding 1: ANE memory-controller byte counters confirm actual execution (arxiv 2608.22110)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2608.22110
- **Date:** August 22, 2026
- **Summary:** Shahir M A's paper "What actually runs" sweeps a 64-shape matrix of LLM primitives and reads the ANE's memory-controller byte counters during inference to establish what actually ran on the engine vs. falling back to CPU. Key finding: placement is a property of how a computation is expressed, not what it computes — a fused RMSNorm is fully ANE-eligible while its arithmetically identical decomposition is CPU-only.
- **Why it matters:** First public technique using ANE memory-controller byte counters as a ground-truth utilization signal; this is the measurement primitive t3rm1nu55-monitorplus would need to implement real ANE activity detection.

### Finding 2: Comprehensive ANE architecture reverse-engineering reference (arxiv 2606.22283)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 2026
- **Summary:** Spencer Bryngelson's "Apple Neural Engine: Architecture, Programming, and Performance" documents the ANE via direct measurement on Apple silicon and static analysis of the private runtime, compiler, kernel driver, and firmware. Covers the datapath and roofline, the dispatch route below CoreML, the compiler and on-disk program format, weight-compression, kernel driver, firmware, and command protocol.
- **Why it matters:** Most comprehensive public ANE internals reference to date; the kernel driver section may expose counter register locations adjacent to the command protocol.

### Finding 3: ANEForge — Python direct ANE programming without CoreML (arxiv 2606.17090)
- **Source:** arXiv / GitHub (github.com/sbryngelson/ANEForge)
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 2026
- **Summary:** Spencer Bryngelson's ANEForge is a Python package (also on PyPI) that compiles a lazy tensor graph of 58 fused operators into a single ANE program dispatched directly through the ANE daemon and kernel-driver stack, bypassing CoreML entirely. A small fused program completes in ~90µs, near the 70µs per-program dispatch floor; supports both inference and training (forward + backward + optimizer) on-device.
- **Why it matters:** The most accessible direct ANE programming interface yet published; the daemon/driver dispatch path it exercises is the same one where memory-controller byte counters (Finding 1) would be readable.

### Finding 4: Orion — end-to-end LLM runtime on ANE with 20-constraint MIL catalog (arxiv 2603.06728)
- **Source:** arXiv / GitHub (github.com/mechramc/Orion)
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026 (missed in seed — published before last sweep date but not captured)
- **Summary:** Orion is the first open end-to-end system combining direct ANE execution, a compiler pipeline, and stable multi-step training with checkpoint resume, bypassing CoreML via Apple's private `_ANEClient`/`_ANECompiler` APIs. It extends maderix's prior characterization with a catalog of 20 restrictions on MIL IR programs, memory layout, compilation limits, and numerical behavior.
- **Why it matters:** The 20-constraint catalog is a precise enumeration of what the ANE will and will not execute — essential context for any model that aims to guarantee ANE placement rather than silent CPU fallback.

### Finding 5: AMX dual-block architecture characterization on M1 (arxiv 2606.25426)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 24, 2026
- **Summary:** Deyvik Bhan (Georgia Tech) characterizes M1 AMX throughput via microbenchmark in "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX." Key structural finding: M1 has two on-chip AMX blocks; the inner loop is load-issue bound at 610–680 GFLOPS per thread when any operand load interleaves with the FMA32 stream, under half the load-free rate. Gains over Accelerate come from fine multi-thread panels and pre-packing weights, not a faster inner loop.
- **Why it matters:** Confirms M1 two-AMX-block topology and establishes the load-issue bound as the correct throughput model for AMX utilization inference; useful when designing AMX activity proxies based on memory bandwidth rather than direct counters.

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
