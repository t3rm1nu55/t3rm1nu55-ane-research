# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-06 — sweep (3 findings)

### Finding 1: Orion — first open system to characterize and program the ANE end-to-end
- **Source:** arXiv / GitHub (mechramc/Orion)
- **URL:** https://arxiv.org/abs/2603.06728 · https://github.com/mechramc/Orion
- **Date:** 2026-03-06
- **Summary:** Academic paper plus MIT-licensed runtime that bypasses CoreML entirely via `_ANEClient` and `_ANECompiler` private APIs. The authors catalogued 20 ANE execution constraints (14 newly discovered MIL IR, memory, and I/O rules) and document that each process is limited to ~119 `ANECCompile()` calls before silent failure — an internal software counter. On M4 Max, deep operation graphs (16–64 ops) achieve 94% ANE utilisation; the paper also demonstrates a weight-patching trick that cuts per-step recompilation from 4,200 ms to 494 ms (8.5×).
- **Why it matters:** The 20-constraint catalog and `_ANECompiler` symbol are the deepest public description of ANE programmable surface to date; the per-process compilation counter is the first documented software-visible ANE state counter.

### Finding 2: maderix Part 3 — training a transformer on the ANE, plus new open-source ANE code repo
- **Source:** maderix Substack / GitHub (maderix/ANE)
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b · https://github.com/maderix/ANE
- **Date:** 2026-03-07
- **Summary:** Third instalment of the maderix M4 ANE series demonstrates full forward pass, backward pass, gradient computation, and Adam optimizer updates for a 109M-parameter transformer trained directly on the ANE with no CoreML or Metal. The companion GitHub repo (`maderix/ANE`) ships working Objective-C code using the same `_ANEClient` private API surface documented in Parts 1–2.
- **Why it matters:** The code repo is the most accessible public implementation of direct ANE dispatch; the training result confirms the ANE can sustain stateful multi-step execution, which is relevant for any future ANE activity-inference approach.

### Finding 3: Efficient MoE LLM Inference with Apple Silicon NPUs (arXiv 2604.18788)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2604.18788
- **Date:** 2026-04-22
- **Summary:** Post-sweep paper characterising ANE behaviour for Mixture-of-Experts LLM inference; reports the M2 ANE as a 16-core unit delivering up to 15.8 TFLOPS FP16 and documents per-expert dispatch patterns that affect ANE utilisation under sparse activation.
- **Why it matters:** Confirms cross-chip ANE core-count figures (M1=16, M2=16) that will inform any utilisation-fraction inference model in t3rm1nu55-monitorplus.

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
