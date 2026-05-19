# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-19 — sweep (4 findings)

### Finding 1: "Orion" arXiv paper — first formal ANE characterization via private APIs
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026
- **Summary:** First academic paper to characterize the ANE by bypassing CoreML entirely via private `_ANEClient` and `_ANECompiler` APIs. Key measured result: deep operation graphs (16–64 ops) achieve 94% ANE utilization at 19 TFLOPS FP16 on M4. No hardware counter API is exposed; utilization is inferred from throughput timing of graph execution at the ANECompiler boundary. Paper formalizes the maderix Substack series and is accompanied by a companion GitHub repo.
- **Why it matters:** Definitive current reference for ANE performance characterization — confirms the counter gap while quantifying what *is* measurable via private API timing.

### Finding 2: maderix/ANE — working direct-ANE training implementation
- **Source:** GitHub
- **URL:** https://github.com/maderix/ANE
- **Date:** Active; ~42 commits, created ~March 2026
- **Summary:** Open-source implementation of direct ANE access for forward and backward passes on 109M–596M parameter transformers, using reverse-engineered `_ANEClient`, `_ANECompiler`, and `_ANEInMemoryModelDescriptor`. Achieves ~5–9% effective utilization under training load with "significant engineering challenges remaining". Documents a ~119 compile limit per process (resource leak in ANE compiler) and confirms SDPA causal masking is unsupported in hardware.
- **Why it matters:** New tracked source with a live codebase; the 5–9% utilization ceiling under real workloads establishes the gap between theoretical peak and observed behavior, directly relevant to IOReport power-based utilization inference.

### Finding 3: ClF3 blog — M3/M4 PMU ESR register encoding change
- **Source:** ClF3's blog
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** 2026 (exact date not confirmed; appeared in recent search results)
- **Summary:** Documents that M3 and M4 PMU Event Status Registers (ESR) are 64-bit with 16 bits per event slot, compared to 8 bits per event on M1/M2. The kpep plist files on disk reflect this change (as4.plist vs a14.plist etc.) but the difference requires explicit handling when parsing event bitmasks across chip generations.
- **Why it matters:** Directly actionable for the kperf sidecar — multi-generation counter code must branch on chip family when reading/writing event configuration registers or it silently programs wrong events on M3+.

### Finding 4: arXiv:2604.18788 — MoE LLM inference on Apple Silicon NPUs
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2604.18788
- **Date:** April 2026
- **Summary:** Introduces NPUMoE, a runtime that offloads Mixture-of-Experts expert weights to the ANE while keeping dynamic routing on CPU/GPU. Uses offline calibration (not hardware counters) to estimate expert capacity and popularity; achieves 1.32×–5.55× latency reduction on M-series devices.
- **Why it matters:** Adds MoE workloads to the set of known patterns that drive ANE power consumption; confirms that state-of-art ANE scheduling still relies on offline calibration rather than real-time counters, reinforcing the open research gap.

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
