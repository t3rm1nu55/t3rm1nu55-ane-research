# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-09-11 — sweep (4 findings)

### Finding 1: sbryngelson/ane-guide — Systematic public documentation of the full ANE stack
- **Source:** arXiv + GitHub (Spencer H. Bryngelson et al.)
- **URL:** https://arxiv.org/abs/2606.22283 / https://github.com/sbryngelson/ane-guide
- **Date:** June 2026
- **Summary:** "Apple Neural Engine: Architecture, Programming, and Performance" is the most comprehensive public analysis of ANE internals to date. It documents the datapath and roofline, the dispatch route that bypasses CoreML entirely, the compiler and on-disk program format, and — critically — the kernel driver, firmware, and command protocol. Every claim is marked as measured, decompile-derived, or predicted.
- **Why it matters:** Documents the ANE kernel driver + firmware command protocol, which is the prerequisite foundation for any future ANE counter/utilization surface in t3rm1nu55-monitorplus. Add to tracked references; read before any ANE telemetry work.

### Finding 2: mechramc/Orion + arxiv 2603.06728 — First open end-to-end direct-ANE system
- **Source:** arXiv + GitHub (mechramc)
- **URL:** https://arxiv.org/abs/2603.06728 / https://github.com/mechramc/Orion
- **Date:** March 2026
- **Summary:** Orion is the first published open system for direct ANE execution, training, and inference that bypasses CoreML using private `_ANEClient`/`_ANECompiler` APIs. The paper catalogs 20 restrictions on MIL IR programs (14 previously undocumented), covering memory layout, compilation limits, and numerical constraints.
- **Why it matters:** The 14 new MIL IR constraints narrow the space of legal ANE programs — essential context for any attempt to instrument ANE dispatch paths in the monitoring sidecar.

### Finding 3: maderix/ANE — New GitHub repo; full transformer training on ANE including backward pass
- **Source:** GitHub (maderix) + Substack Part 3
- **URL:** https://github.com/maderix/ANE / https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** 2026 (post April 2026)
- **Summary:** A new companion repository to the maderix Substack series. Demonstrates full transformer training on the M4 ANE (forward pass, backward pass, Adam optimizer on 109M parameters) via reverse-engineered private APIs, with INT8 W8A8 quantization (1.88× throughput). Reports peak ANE throughput at 18.6 TFLOPS (FP16) / 35.1 TFLOPS (INT8), but observes actual training utilization is only ~5–9% of peak. No counter access — benchmarking only. Also exposes SRAM bandwidth probing utilities.
- **Why it matters:** The SRAM probing utilities and empirical utilization gap (~5–9% of peak) are concrete inputs for the power-indirection heuristic in t3rm1nu55-monitorplus; the INT8 throughput figures update the best public characterization of ANE peak.

### Finding 4: arxiv 2606.25426 — Direct AMX programming bypasses Accelerate; reveals dual-block M1 structure
- **Source:** arXiv (Deyvik Bhan, Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" is the first published work demonstrating direct M1 AMX programming without going through Accelerate. Key finding: the M1 AMX inner loop is load-issue bound at ~610–680 GFLOPS single-threaded (under half the load-free rate); speedup over Accelerate comes from filling the M1's second AMX block via multi-thread panels. Achieves 1.58× geometric mean over Accelerate's fastest path.
- **Why it matters:** Confirms two AMX blocks exist per M1 and that they are independently addressable; this is new public microarchitecture detail that matters for AMX event-discovery work (kperf events 0x96/0x9b constrained to counter 7 may map to the AMX FMA stream).

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
