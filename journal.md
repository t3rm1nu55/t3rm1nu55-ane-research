# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-04-25 — sweep (2 findings)

*Note: this is the first executed sweep. The 2026-04-07 entry was the repository seed, not a real sweep run. Both findings below were published in March 2026 and were missed by the seed.*

### Finding 1: Orion — first open runtime for ANE training and inference, bypassing CoreML

- **Source:** arXiv / GitHub mechramc/Orion
- **URL:** https://arxiv.org/abs/2603.06728 / https://github.com/mechramc/Orion
- **Date:** March 6, 2026
- **Summary:** Orion is the first open end-to-end system that trains and runs LLMs directly on the ANE by bypassing CoreML via the private `_ANEClient` and `_ANECompiler` APIs. It measures ANE utilization as time-in-`orion_eval` vs total wall time — a timing-based proxy, not a hardware counter. The paper catalogs 20 ANE constraints (14 newly discovered), achieves 170+ tokens/s GPT-2 124M inference on M4 Max, and reduces per-step training recompilation from 4,200 ms to 494 ms via weight-patching. It also confirms the ANE compiler silently rejects compilation requests after ~119 per process.
- **Why it matters:** Confirms timing-fraction is the current state-of-the-art ANE utilization metric; no hardware counter surface was found. The constraint catalog and `_ANEClient` call sequence are the most complete public record to date.

### Finding 2: M5 GPU Neural Accelerators expose neural acceleration via public Metal 4 Tensor APIs

- **Source:** Apple developer docs / tzakharko benchmark / MLX issue #2693
- **URL:** https://tzakharko.github.io/apple-neural-accelerators-benchmark/ / https://github.com/ml-explore/mlx/issues/2693
- **Date:** October 2025 (M5 base launch); March 2026 (M5 Pro/Max)
- **Summary:** M5 GPUs embed a dedicated Neural Accelerator (matrix-multiply unit) per GPU core achieving 1024 FP16 FLOPS/core/cycle, accessible via the fully public Metal 4 Tensor APIs and Metal Performance Primitives — unlike the ANE's private `_ANEClient`. Metal's existing `MTLCounterSet` performance sampling infrastructure is present on M5 hardware and may already surface Neural Accelerator utilization as a standard GPU counter. The tzakharko benchmark characterizes the M5/A19 Neural Accelerator microarchitecture via black-box throughput tests.
- **Why it matters:** First generation where GPU-integrated neural acceleration is reachable via a public API; whether `MTLCounterSet` exposes a Neural Accelerator counter on M5 is now an actionable investigation for t3rm1nu55-monitorplus.

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
