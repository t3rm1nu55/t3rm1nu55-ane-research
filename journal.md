# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-27 — sweep (3 findings)

### Finding 1: M5 GPU Neural Accelerators — per-core accelerator with Metal utilization counter

- **Source:** Apple Developer Tech Talks; tzakharko benchmark (`tzakharko.github.io/apple-neural-accelerators-benchmark/`); macgpu.com blog (2026-04-25)
- **URL:** https://developer.apple.com/videos/play/tech-talks/111432/ · https://tzakharko.github.io/apple-neural-accelerators-benchmark/
- **Date:** M5 MacBook launch March 2026; benchmark post published April–May 2026
- **Summary:** The M5/A19 generation introduces a new "Neural Accelerator" inside each GPU core, distinct from the traditional ANE, and programmable via Metal 4 Tensor APIs. Apple's Metal debugger and Metal system trace expose a **Neural Accelerator utilization counter** for these units; a developer tech talk explicitly shows utilization rising from 0 % to >50 % when TensorOps are used. Taras Zakharko has published an open microbenchmark characterizing the A19/M5 GPU Neural Accelerators' throughput and behavior. MLX is tracking Metal 4 Tensor support (ml-explore/mlx#2693).
- **Why it matters:** MTLCounterSampleBuffer is a public, programmatic API — if the Neural Accelerator counter is in a device's counter set, t3rm1nu55-monitorplus could query it directly without any private framework, giving the per-chip "neural accelerator utilization %" metric that has been the open problem. **This is the highest-priority lead in this research repo.**

### Finding 2: Orion — peer-reviewed open-source ANE compiler for LLM inference + training (arxiv 2603.06728)

- **Source:** arXiv (new source, not previously tracked)
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** Submitted March 6, 2026 (missed by initial seed)
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" by Ramchand Kumaresan presents the first open end-to-end system for ANE LLM execution, bypassing CoreML entirely via private `_ANEClient`/`_ANECompiler` APIs. It provides a consolidated catalog of 20 ANE programming constraints — 14 newly discovered MIL IR restrictions — plus memory layout requirements and numerical behaviors not previously documented.
- **Why it matters:** Authoritative academic characterization of the private API surface; the 14 new MIL IR restrictions refine the model of what the ANE hardware enforces, which could inform how monitorplus identifies active ANE dispatch patterns.

### Finding 3: maderix — ANE Part 3 (Training) + open-source `maderix/ANE` repo

- **Source:** maderix Substack (tracked); new GitHub repo `maderix/ANE` (new source)
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b · https://github.com/maderix/ANE
- **Date:** March 7, 2026 (missed by initial seed; initial seed only referenced Part 2)
- **Summary:** Part 3 demonstrates full backpropagation on the ANE — a 109 M-parameter transformer trained from scratch at 1.78 TFLOPS sustained (9.3 ms/step), using reverse-engineered `_ANEClient` and `_ANECompiler` private APIs. All code is publicly available at `github.com/maderix/ANE` with working forward pass, backward pass, gradient computation, and Adam optimizer running natively on the ANE.
- **Why it matters:** The `maderix/ANE` repo is now the most complete public reference implementation of `_ANEClient` private API invocation; any effort to hook ANE dispatch for utilization measurement in monitorplus can treat it as the ground-truth usage pattern.

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
