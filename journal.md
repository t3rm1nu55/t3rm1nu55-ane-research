# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-29 — sweep (2 findings)

### Finding 1: Orion — First open-source direct ANE compiler and runtime (missed by initial seed)
- **Source:** arXiv + GitHub (mechramc/Orion)
- **URL:** https://arxiv.org/abs/2603.06728 · https://github.com/mechramc/Orion
- **Date:** March 6, 2026 (published before April 7 cutoff but absent from initial seed)
- **Summary:** Orion is the first open-source end-to-end system combining direct ANE execution, a compiler pipeline (lowering through 5 passes to ANE-native MIL IR), and stable multi-step transformer training — bypassing CoreML entirely via Apple's private `_ANEClient` and `_ANECompiler` APIs. The "94% ANE utilization" figure cited in the paper is a throughput proxy derived from wall-clock benchmarking at saturating graph depth, not a hardware counter. The open-source code now gives a working reference implementation of the full `_ANEClient` dispatch path.
- **Why it matters:** The `_ANEClient` interface is the most likely host-side vector for any undocumented ANE telemetry query; Orion makes that surface inspectable in working code for the first time.

### Finding 2: NPUMoE — ANE energy measurement methodology on M2 Max/Ultra
- **Source:** arXiv 2604.18788
- **URL:** https://arxiv.org/abs/2604.18788
- **Date:** April 20, 2026
- **Summary:** "Efficient Mixture-of-Experts LLM Inference with Apple Silicon NPUs" (Benazir & Lin) presents NPUMoE, a runtime that offloads dense MoE sub-graphs to the ANE via CoreML while falling back to CPU/GPU for dynamic routing. Evaluation on M2 Max and M2 Ultra shows 1.81×–7.37× energy efficiency improvement attributed to ANE, measured via wall-clock time combined with powermetrics/IOReport energy deltas — the current best-practice methodology short of hardware counters. No new counter APIs are exposed; this is IOReport + timing.
- **Why it matters:** Confirms that IOReport energy delta ÷ wall-clock interval remains the state-of-the-art ANE utilization proxy; no counter API breakthrough has emerged as of April 2026.

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
