# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-05 — sweep (4 findings)

> Note: findings 1–3 were published before the last-checked date (2026-04-07) but were absent from the initial seed. Finding 4 is post-cutoff. All four clear the relevance filter.

### Finding 1: Apple M5 GPU Neural Accelerators — new programmable hardware surface via Metal 4 TensorOps

- **Source:** Apple Newsroom / Apple Machine Learning Research
- **URL:** https://www.apple.com/newsroom/2025/10/apple-unleashes-m5-the-next-big-leap-in-ai-performance-for-apple-silicon/ ; https://machinelearning.apple.com/research/exploring-llms-mlx-m5
- **Date:** October 2025 (base M5); March 3, 2026 (M5 Pro/Max)
- **Summary:** The M5 introduces dedicated Neural Accelerators embedded in every GPU core — separate from the traditional ANE — each capable of 1,024 FP16 FMAs/cycle and programmable directly via Metal 4 TensorOps. The M5 Max 40-core GPU aggregates to ~70 TFLOPS FP16 via these units. This is a qualitatively new hardware surface that did not exist in M4. Whether IOReport's "Energy Model" exposes a distinct power channel for the M5 GPU Neural Accelerators (analogous to the existing ANE channel) is currently unknown and unconfirmed in any public source.
- **Why it matters:** If IOReport gains a GPU Neural Accelerator energy channel on M5, it directly extends the power-indirection ANE-detection pattern already implemented in the main project to a new accelerator class.

### Finding 2: Orion — first end-to-end ANE system paper characterises 20 hardware constraints (14 previously undocumented)

- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" (Ramchand Kumaresan) is the first academic paper to systematically document ANE compiler constraints at the MIL IR level, cataloguing 20 restrictions — 14 previously undocumented — on graph shape, memory layout, numerical behaviour, and compilation limits. It bypasses CoreML entirely via the same `_ANEClient`/`_ANECompiler` private APIs as maderix's work, and adds a checkpoint-resume training loop. No new hardware performance counters are disclosed; measurement remains IOReport energy.
- **Why it matters:** The 14 newly documented ANE MIL constraints give the clearest public picture of what computation shapes actually land on the ANE vs. falling back to GPU/CPU — critical for interpreting IOReport energy deltas as a utilization proxy.

### Finding 3: maderix Part 3 + ANE code repo — confirmed private API training + first M5 ANE data point

- **Source:** maderix Substack / GitHub
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b ; https://github.com/maderix/ANE
- **Date:** March 7, 2026
- **Summary:** Part 3 of the maderix M4 ANE series covers full transformer training (109M and 596M parameter models) on the ANE via `_ANEClient`. The accompanying code repo (`maderix/ANE`) exposes the full private API surface used, including `_ANEInMemoryModelDescriptor`. A contributor provided the first M5 data point: same H16G family as M4, same weight-baking limitation, same QoS behaviour — M5's ANE appears unchanged from M4. Training achieves only 5–9% of ANE peak (element-wise ops still fall back to CPU), confirming utilization remains unmeasured by any counter.
- **Why it matters:** `maderix/ANE` is now a live code reference for `_ANEClient`/`_ANECompiler` symbol resolution; the api_exploration.m file is the most current public catalogue of ANE private API entry points. The M5 ANE sameness finding means the existing IOReport "Energy Model" channel approach carries forward unchanged to M5.

### Finding 4: NPUMoE (arXiv 2604.18788) — first systematic characterisation of ANE MoE dispatch patterns, 1.8–7.4× energy efficiency gains measured via IOReport

- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2604.18788
- **Date:** April 20, 2026 *(post-cutoff)*
- **Summary:** "Efficient Mixture-of-Experts LLM Inference with Apple Silicon NPUs" (Benazir & Lin) introduces NPUMoE, a runtime that offloads the dense, static expert sub-graphs of MoE LLMs to the ANE while routing dynamic operations (top-k, scatter/gather) through CPU/GPU fallback. Energy efficiency is measured via IOReport power sampling, demonstrating 1.81–7.37× improvement and 1.32–5.55× latency reduction over GPU-only paths. The paper is the first to correlate specific graph sub-structures with ANE dispatch/energy signatures at the MoE expert granularity.
- **Why it matters:** The NPUMoE dispatch taxonomy (which ops reliably land on ANE vs. not) is directly usable as a calibration guide for the power-indirection utilization model in t3rm1nu55-monitorplus.

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
