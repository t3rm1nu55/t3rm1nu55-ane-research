# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-18 — sweep (3 findings)

### Finding 1: Orion — first open system for direct ANE programming and LLM training
- **Source:** arXiv (Ramchand Kumaresan)
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026 (missed by initial seed)
- **Summary:** Orion is the first open end-to-end system that bypasses CoreML entirely via Apple's private `_ANEClient` and `_ANECompiler` APIs, adding a compiler pipeline for direct ANE graph execution and stable multi-step LLM training. Building on maderix's characterization work, it catalogues 20 ANE MIL IR restrictions, identifies a 32 MB SRAM performance cliff, and achieves 8.5× reduction in recompilation time and 170+ tokens/s GPT-2 inference on M4 Max. All utilization measurements are timing-derived; no hardware counter surface was discovered.
- **Why it matters:** Authoritatively confirms ANE exposes zero hardware counters — IOReport energy-power inference remains the only real-time monitoring axis, which is exactly what t3rm1nu55-monitorplus v1 uses.

### Finding 2: M3/M4 PMU ESR registers are 64-bit with 16-bit per-event encoding
- **Source:** clf3's blog
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** Unknown (2025–early 2026; not in tracked references)
- **Summary:** Documents a breaking architectural difference: on M3 and M4, ESR registers are 64-bit and each event occupies 16 bits, whereas M1/M2 use a narrower layout. The M4 kpep event database is `as4.plist`. Provides a working kperf/kpc implementation that handles the generation-specific encoding. Complements bugsiki's constraint-rule analysis with a concrete per-generation ESR width table.
- **Why it matters:** The t3rm1nu55-monitorplus kperf privileged sidecar must detect chip generation and apply the correct ESR encoding at initialisation; using M1/M2 encoding on M3/M4 will silently misconfigure counter slots.

### Finding 3: NPUMoE — ANE dispatch overhead characterised as a scheduling bottleneck
- **Source:** arXiv (2604.18788)
- **URL:** https://arxiv.org/abs/2604.18788
- **Date:** April 20, 2026
- **Summary:** NPUMoE offloads dense MoE transformer layers to Apple's ANE while keeping dynamic expert routing on CPU/GPU. The paper characterises ANE dispatch overhead as a dominant latency term for small expert kernels and establishes that the ANE queue depth is bounded at 127 in-flight evaluation requests. This is the first post-seed paper to quantify dispatch queue saturation as a utilization proxy.
- **Why it matters:** The 127-evaluation queue limit is useful context for any future ANE activity detection based on dispatch-queue depth monitoring rather than power sampling.

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
