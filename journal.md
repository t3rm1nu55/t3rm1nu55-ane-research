# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-12 — sweep (5 findings)

### Finding 1: Apple Neural Engine — Architecture, Programming, and Performance (arxiv 2606.22283)
- **Source:** arXiv / Spencer H. Bryngelson (Georgia Tech); companion repo: [sbryngelson/ane-guide](https://github.com/sbryngelson/ane-guide)
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 21, 2026
- **Summary:** Comprehensive reverse-engineered guide to ANE internals based on direct hardware measurement and static analysis of the private runtime, compiler, kernel driver, and firmware. Documents the datapath and roofline bounding throughput and energy, the dispatch route below CoreML, the compiler and on-disk program format, the weight-compression scheme, and the kernel driver/firmware command protocol.
- **Why it matters:** The most comprehensive public ANE internals reference ever published; the documented roofline methodology directly enables an IOReport-energy-delta–based utilization estimator calibrated against the measured peak — the most tractable path to "ANE % utilization" in t3rm1nu55-monitorplus.

### Finding 2: ANEForge — Python library for direct ANE dispatch below CoreML (arxiv 2606.17090)
- **Source:** arXiv / Spencer H. Bryngelson (Georgia Tech); GitHub: [sbryngelson/ANEForge](https://github.com/sbryngelson/ANEForge)
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 12, 2026
- **Summary:** Python package that compiles lazy tensor graphs into ANE programs (58 fused operators, 19 bridge operators) and dispatches through the same ANE daemon and kernel-driver stack as Apple's internal framework, entirely bypassing CoreML. Achieves ~90µs round-trip near the engine's 70µs per-program dispatch floor; ResNet-18 forward pass completes in 0.33ms.
- **Why it matters:** First open-source tool that documents and exercises the private ANE dispatch path and driver interface; the driver ABI it exposes is the same layer Apple's internal profiling tools sit on, making it the best current reference for what runtime observability exists below CoreML.

### Finding 3: M1 AMX microbenchmarked; M4+ migrated to ARM SME (arxiv 2606.25426)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** "Above the Inner Loop" measures M1 AMX throughput at 610–680 GFLOPS under load-issue constraints, reveals M1 has two on-chip AMX blocks exploitable via multi-thread packing, and confirms M4+ chips replaced proprietary AMX with ARM Scalable Matrix Extension (SME) — a publicly documented standard with a defined PMU counter interface.
- **Why it matters:** M4+ migration to ARM SME means matrix-unit utilization is measurable via standard ARM PMU events on those chips; on M1–M3, the quantified throughput ceiling enables a power-proxy utilization estimator using kperf cycle counts against the documented peak.

### Finding 4 (pre-cutoff, missed in seed): Orion — end-to-end ANE LLM training without CoreML (arxiv 2603.06728)
- **Source:** arXiv / Ramchand Kumaresan; GitHub: [mechramc/Orion](https://github.com/mechramc/Orion)
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026
- **Summary:** First end-to-end system for LLM training and inference directly on ANE using Apple's private `_ANEClient` and `_ANECompiler` APIs, without CoreML. Catalogs 20 ANE constraints including 14 previously undocumented ones. Achieves 170+ tokens/s for GPT-2 124M inference on M4 Max; trains a 110M-parameter transformer in 22 minutes.
- **Why it matters:** Documents previously unknown ANE driver constraints and demonstrates the full private API path below CoreML; complements ANEForge and Finding 1 as a reference for what runtime instrumentation points exist in the dispatch stack.

### Finding 5 (pre-cutoff, missed in seed): maderix/ANE adds M5 support with training telemetry (GitHub)
- **Source:** GitHub — [maderix/ANE](https://github.com/maderix/ANE)
- **URL:** https://github.com/maderix/ANE/commit/893f58e
- **Date:** March 2, 2026
- **Summary:** Merged PR "m5-maximized" (contributor: m0at) adding M5 support to maderix's reverse-engineered ANE training project, described as "ANE probe tests + training telemetry for M5 optimization." Extends the direct-ANE training work to Apple's newest chips.
- **Why it matters:** First public mention of timing/probe telemetry specifically for the M5 ANE; the "probe tests" may reveal M5-specific timing measurement points worth examining for counter discovery.

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
