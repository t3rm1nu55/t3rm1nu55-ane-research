# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-17 — sweep (3 findings)

### Finding 1: maderix Part 3 — Training on the M4 ANE (missed from initial seed)
- **Source:** maderix Substack
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** February–March 2026 (published before repo creation; absent from initial seed)
- **Summary:** Third post in the maderix M4 ANE reverse-engineering series. Demonstrates full transformer training (forward pass, backward pass, Adam optimizer, 109 M parameters) on hardware Apple designed exclusively for inference. Characterises the ANE compiler's hard ceiling of ~119 compilations per process — after which `ANECCompile()` silently fails — and documents the weight-blob format required to re-parameterize existing program objects. Introduces an `exec()`-restart workaround and previews delta compilation as a permanent fix (formalised in the Orion system, Finding 2 below).
- **Why it matters:** The 119-compile ceiling and the weight-blob re-parameterisation requirement constrain any long-running ANE utilisation monitor that hooks into `ANECCompile`; understanding the limit is a prerequisite for any v2 ANE hook in t3rm1nu55-monitorplus.

### Finding 2: Orion — First open end-to-end ANE runtime with constraint catalogue (missed from initial seed)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026 (published before repo creation; absent from initial seed)
- **Summary:** Orion is the first open system to combine direct ANE execution (bypassing CoreML via private `_ANEClient`/`_ANECompiler` APIs), a compiler pipeline, and stable multi-step training with checkpoint resume. The paper extends the documented set of ANE compilation constraints from 6 to 20 — 14 previously undocumented — and characterises the 32 MB on-chip SRAM performance cliff (throughput drops sharply when an operation graph exceeds it) and a ~0.095 ms per-graph dispatch overhead. Delta compilation (`ANECCompile()` is replaced by reloading existing program objects with updated weight files) eliminates the 119-compilation ceiling.
- **Why it matters:** The 20-constraint catalogue, SRAM cliff threshold, and dispatch timing are the most precise public characterisation of ANE execution limits to date; any future ANE telemetry or utilisation-hook work in t3rm1nu55-monitorplus should be validated against these constraints.

### Finding 3: NPUMoE — Apple Silicon NPU offloading with documented energy attribution (post-cutoff)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2604.18788
- **Date:** April 20, 2026
- **Summary:** NPUMoE is a runtime inference engine that offloads the static, dense portion of Mixture-of-Experts computation to the Apple Silicon NPU while routing dynamic operations through CPU/GPU. Evaluated on three MoE LLMs across four long-context workloads on M-series devices, it reports 1.32×–5.55× latency reduction and 1.81×–7.37× energy efficiency improvement over CPU/GPU baselines.
- **Why it matters:** The reported energy efficiency gains implicitly validate IOReport Energy Model as a trustworthy ground truth for ANE load attribution; if energy were not correctly attributed to the ANE by IOReport the efficiency ratios would be meaningless — confirming IOReport is the right primitive for t3rm1nu55-monitorplus's ANE energy inference path.

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
