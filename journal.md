# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-01 — sweep (3 findings)

Two of the three findings below are **catch-ups**: they were published in March 2026 but were not captured in the 2026-04-07 initial seed. One finding (arXiv 2604.18788) is genuinely new since the last sweep.

### Finding 1: Orion — first open ANE end-to-end system with `_ANEClient` compiler pipeline
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** 2026-03-06 (catch-up; missed by initial seed)
- **Summary:** Orion is the first open system that combines direct ANE execution, a custom compiler pipeline, and multi-step training in a single runtime, bypassing CoreML entirely via `_ANEClient` and `_ANECompiler` private APIs. The paper catalogs 20 restrictions on MIL IR programs and memory layout — 14 of which were previously undocumented — and identifies a per-process ANE compilation limit of approximately 119 invocations before silent failures occur. Tensor I/O uses IOSurface-backed shared memory in a fixed `[1, C, 1, S]` fp16 layout, enabling zero-copy CPU↔ANE transfer.
- **Why it matters:** The IOSurface layout and the per-process compilation counter are the closest thing yet to an indirect ANE activity metric; these details directly inform any future t3rm1nu55-monitorplus effort to move beyond IOReport power-gating as the sole ANE observable.

### Finding 2: maderix Part 3 — transformer training on M4 ANE via reverse-engineered private APIs
- **Source:** maderix Substack
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** 2026-03 (catch-up; initial seed referenced Parts 1–2 only)
- **Summary:** Part 3 of the maderix ANE series demonstrates a full training loop — forward pass, backward pass, gradient computation, Adam updates — running on the M4 ANE for a 109M-parameter transformer, using the same `_ANEClient`/`_ANECompiler` path established in Parts 1–2. Accompanying code is published at https://github.com/maderix/ANE.
- **Why it matters:** Confirms that ANE utilization during training workloads (not just inference) is now a real scenario; the `maderix/ANE` repo is a new tracked codebase that could surface further API/counter discoveries.

### Finding 3: arXiv 2604.18788 — NPUMoE: MoE LLM inference on Apple Silicon ANE with energy benchmarks
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2604.18788
- **Date:** 2026-04-20 (new since last sweep)
- **Summary:** NPUMoE offloads the dense/static computation of Mixture-of-Experts models to the Apple Silicon ANE, achieving 1.32×–5.55× latency reduction and 1.81×–7.37× energy efficiency improvement on M2 Ultra/Max versus CPU-only baselines. The paper measures a 16-core M2 ANE at 15.8 TFLOPS FP16 and uses IOReport (not powermetrics) for per-interval energy accounting, treating the ANE energy channel as ground truth.
- **Why it matters:** Independently validates that IOReport's ANE energy channel is the right measurement primitive, and provides a published throughput/energy characterization of M2 ANE that can calibrate our per-interval power-inference heuristics.

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
