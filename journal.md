# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-03 — sweep (5 findings)

### Finding 1: Comprehensive public ANE reverse engineering — architecture, dispatch, firmware, command protocol
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 2026
- **Summary:** Spencer Bryngelson's "Apple Neural Engine: Architecture, Programming, and Performance" documents the full ANE stack from direct measurement on Apple Silicon and static analysis of private runtime, compiler, kernel driver, and firmware: datapath and roofline, below-CoreML dispatch route, on-disk program format, weight-compression scheme, and command protocol. This is the most comprehensive public treatment of the ANE hardware and software stack to date.
- **Why it matters:** Command protocol and kernel driver documentation may reveal a utilization measurement or counter surface not previously accessible; highest-priority follow-up paper for v2 ANE work in t3rm1nu55-monitorplus.

### Finding 2: ANEForge — open Python direct-ANE dispatch without CoreML
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 2026
- **Summary:** ANEForge is a Python package that compiles a lazy tensor graph (58 fused operators + 19 native bridge operators) into a single ANE program dispatched via private symbols, bypassing CoreML entirely; it supports native fused attention, int8/int4/sparse weight streaming, and full forward+backward training passes through the same ANE daemon and kernel-driver stack Apple uses internally.
- **Why it matters:** First fully open, non-CoreML ANE dispatch framework; the 58 fused operator surface and private dispatch calling convention are useful references for any future ANE utilization probe.

### Finding 3: M1 AMX dual-block topology and load-issue-bound inner loop characterized
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" discovers that M1 contains a second on-chip AMX block and that the fp32 GEMM inner loop is load-issue-bound (~610–680 GFLOPS vs. peak rate); fine multi-thread panels filling both blocks achieve a geometric mean 1.58× speedup over BNNSMatMul with bit-exact results.
- **Why it matters:** First public documentation of M1's dual AMX block count and thread-to-block mapping; foundational topology knowledge for any kperf AMX event discovery or AMX utilization inference.

### Finding 4: maderix/ANE repo — first open codebase for direct ANE training dispatch
- **Source:** GitHub / maderix Substack (Part 3)
- **URL:** https://github.com/maderix/ANE
- **Date:** 2026 Q2
- **Summary:** maderix released an open repo implementing transformer training (forward + backward pass, Adam optimizer, up to 596M parameters on Qwen3-0.6B) on the M4 ANE via private `_ANEClient` and `_ANECompiler` APIs without CoreML, accompanying a three-part Substack series. The calling sequence, graph compilation flow, and benchmarks constitute the most detailed public reference for direct ANE hardware access below CoreML.
- **Why it matters:** The `_ANEClient` dispatch sequence here is the clearest existing guide for reaching the ANE programmatically — directly relevant to any future ANE power/utilization probe below the IOReport layer.

### Finding 5: Orion — 14 newly discovered ANE constraints, LLM training/inference framework
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 2026 (predates sweep window; not logged in initial seed)
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" extends the public ANE constraint catalog from 6 to 20 restrictions (14 newly discovered MIL IR, memory, and I/O constraints) and builds an open framework for LLM training and inference directly on the ANE.
- **Why it matters:** The constraint catalog directly informs which operator patterns can and cannot be compiled to ANE — relevant baseline for any future ANE graph dispatch logic or workload detection heuristic.

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
