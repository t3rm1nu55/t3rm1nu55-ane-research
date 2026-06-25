# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-25 — sweep (3 findings)

### Finding 1: Orion — first open end-to-end ANE runtime via _ANEClient (missed by seed)
- **Source:** arXiv / mechramc
- **URL:** https://arxiv.org/abs/2603.06728 · https://github.com/mechramc/Orion
- **Date:** 2026-03-06 (published before sweep window but absent from seed)
- **Summary:** Orion is the first public, open-source system that bypasses CoreML entirely by driving the ANE through Apple's private `_ANEClient` and `_ANECompiler` APIs with a custom MIL IR compiler pipeline. It extends the known ANE constraint catalog to 20 restrictions (14 newly documented), achieves 170+ tokens/s for GPT-2 inference on M4 Max, and demonstrates stable transformer training on ANE hardware. Companion maderix/ANE repo contains the foundational `_ANEClient` API exploration in Objective-C.
- **Why it matters:** This is the most complete public documentation of the private ANE execution surface. The `_ANEClient` call sequence exposed here is the basis for any future "is ANE busy" probe workload or API-hook approach in t3rm1nu55-monitorplus.

### Finding 2: NPUMoE — ANE constraint surface for MoE inference (arXiv 2604.18788)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2604.18788
- **Date:** 2026-04-20
- **Summary:** NPUMoE presents a runtime MoE inference engine that offloads dense, static expert computation to the ANE while routing dynamic operations to CPU/GPU. The paper systematically catalogs three hard ANE constraints that block naïve MoE offloading: ANE requires static tensor shapes at compile time, lacks support for scatter/gather and top-k, and incurs high per-kernel dispatch overhead. Energy measurement uses wall-clock timing against chip TDP baselines, achieving 1.81x–7.37x efficiency gains.
- **Why it matters:** Confirms the ANE's shape-specificity constraint and documents the static-graph compilation requirement — directly relevant to designing a probe workload for binary ANE busy-state detection in t3rm1nu55-monitorplus.

### Finding 3: IOReport ANE channel naming differs on multi-die (Ultra) chips
- **Source:** kennss/SiliconScope — IOReport channel audit
- **URL:** https://github.com/kennss/SiliconScope
- **Date:** ~2026 (exact commit date unconfirmed)
- **Summary:** SiliconScope's `--power-debug` diagnostic, which dumps every IOReport power channel per group, documents that ANE power sits in the "Energy Model" group as `ANE` on single-die chips (M1/M2/M3/M4 base/Pro/Max) but as `ANE0_0`, `ANE0_1` etc. on multi-die Ultra chips. The SiliconScope codebase uses a `strip_die_suffix` pattern to normalise these. A separate diagnostic RFC in the ml-energy/zeus-apple-silicon project confirms the same naming split.
- **Why it matters:** t3rm1nu55-monitorplus's IOReport channel parser may silently miss ANE power on Ultra chips if it matches only `ANE` exactly — should add suffix-tolerant matching (e.g. `starts_with("ANE")`).

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
