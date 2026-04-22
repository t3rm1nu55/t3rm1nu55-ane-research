# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-04-22 — sweep (3 findings)

> Note: all three findings were published in the March 2026 window and were missed by the April 7 seed synthesis. This sweep is a catch-up as well as a current check; no post-April 7 new work found today.

### Finding 1: Orion — first open end-to-end ANE LLM runtime via private APIs
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** 2026-03-06
- **Summary:** Ramchand Kumaresan's "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" is the deepest public technical treatment of direct ANE programming to date. It bypasses CoreML entirely using `_ANEClient` and `_ANECompiler` private APIs, achieves 170+ tokens/s for GPT-2 124M inference on M4 Max, and documents 14 newly discovered MIL IR + memory constraints. Deep operation graphs (16–64 ops) drive 94% ANE utilization (benchmarked, not counter-derived). It also documents that the ANE compiler limits each process to ~119 compilations before silently failing, and shows a delta-compilation bypass.
- **Why it matters:** Orion's documented API surface (`_ANEClient`, `_ANECompiler`, `_ANEInMemoryModelDescriptor`) and its compilation-limit workaround are the current state of the art for any future t3rm1nu55-monitorplus v2 ANE utilization feature — the 94% figure establishes what the hardware can do; the API path shows how to query it.

### Finding 2: maderix ANE Part 3 (Training) + companion GitHub repo
- **Source:** maderix Substack / GitHub
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b  |  https://github.com/maderix/ANE
- **Date:** 2026-03-07 (Part 3 post); repo active through March 2026
- **Summary:** Part 3 of the tracked maderix series (only Part 2 was seeded) cracks the ANE weight blob format, works around the ~119-compilation ceiling via object reuse, and successfully trains a small transformer on the ANE. M5 data points are included, making this the first public performance characterisation for M5 ANE. The companion `maderix/ANE` GitHub repo ships working Objective-C code for `_ANEClient`/`_ANESharedEvents`/`api_exploration.m` — directly inspectable private API surface.
- **Why it matters:** The `api_exploration.m` file in the companion repo is a live reference for `_ANEClient` method signatures; any attempt to build an ANE telemetry shim in the monitorplus sidecar would start here.

### Finding 3: XTC — first cross-platform research harness using kperf/KPep on Apple Silicon
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2512.16512
- **Date:** 2025-12-18
- **Summary:** "XTC: A Research Platform for Optimizing AI Workload Operators" (December 2025, missed by seed) presents a unified scheduling and measurement harness across x86, non-Apple ARM, NVIDIA GPUs, and — for the first time publicly — Apple Silicon CPUs, via Apple's undocumented `KPerf` system interface and the `KPep` database for event translation. The paper claims this makes XTC the first system to programmatically access hardware performance counters on Apple Silicon in a portable, reproducible research context.
- **Why it matters:** XTC's kperf integration is a documented, tested approach to the same private interface our kperf sidecar uses; its KPep event-translation layer and counter-halt/retrieval pattern are worth studying against the ibireme gist reference to see if there are event names or counter groupings XTC uses that we don't yet expose.

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
