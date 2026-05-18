# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-18 — sweep (2 findings)

### Finding 1: NPUMoE — ANE energy efficiency benchmarks for MoE LLM inference (arXiv 2604.18788)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2604.18788
- **Date:** April 20, 2026
- **Summary:** NPUMoE offloads dense, static MoE computations (attention, expert FFNs) to the Apple Neural Engine via offline-compiled Core ML graphs, while keeping dynamic operations (top-k routing, scatter/gather) on CPU/GPU. Evaluated on M2 Max and M2 Ultra, it achieves 1.81x–7.37x energy efficiency improvement and 1.32x–5.55x latency reduction vs. CPU/GPU baselines. No new hardware counters are exposed; utilization is inferred from power/latency measurements.
- **Why it matters:** The offline-calibration + runtime-dispatch split methodology is the most detailed public treatment of how to attribute energy to ANE workloads — useful as a reference for t3rm1nu55-monitorplus's ANE power inference approach.

### Finding 2: macmon v0.7.1 — IOReport channel naming diverges on Ultra chips (`DIE_N_` prefix)
- **Source:** vladkens/macmon
- **URL:** https://github.com/vladkens/macmon/releases/tag/v0.7.1
- **Date:** April 15, 2026
- **Summary:** Fixed CPU usage always reading 0% on M1/M2/M3 Ultra chips. Root cause: multi-die Ultra SoCs expose IOReport CPU-stats channels under a `DIE_N_`-prefixed, `_CPU`-suffixed naming scheme rather than the single-die format. The bug had gone undetected because Ultra chips aren't in the CI matrix.
- **Why it matters:** t3rm1nu55-monitorplus uses the same IOReport access pattern as macmon; any code that looks up CPU stat channels by exact name will silently return zero on Ultra chips. Requires explicit handling of the `DIE_N_` variant.

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
