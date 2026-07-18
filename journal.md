# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-18 — sweep (7 findings)

### Finding 1: Comprehensive ANE architecture reference — "Apple Neural Engine: Architecture, Programming, and Performance"
- **Source:** arXiv (Spencer H. Bryngelson, Georgia Tech) — also at [sbryngelson/ane-guide](https://github.com/sbryngelson/ane-guide) and [ane-guide.readthedocs.io](https://ane-guide.readthedocs.io)
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 21, 2026 (v2: June 27, 2026)
- **Summary:** Reverse-engineered reference documenting ANE datapath, throughput/energy roofline, dispatch route below CoreML, compiler pipeline, on-disk program format, weight-compression scheme, kernel driver, firmware, and command protocol — all derived via direct measurement on Apple silicon plus static decompilation of private runtime and firmware. The most complete public treatment of ANE internals to date.
- **Why it matters:** Documents the exact dispatch path below CoreML that any counter-access or utilization-measurement mechanism would have to use.

### Finding 2: Python direct ANE programming library — ANEForge
- **Source:** arXiv (Spencer H. Bryngelson) — code at [sbryngelson/ANEForge](https://github.com/sbryngelson/ANEForge)
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 2026
- **Summary:** Python package that compiles a lazy tensor graph (58 fused operators, 19 native bridge operators) into ANE programs without CoreML; supports inference and training with int8/int4/sparse weights. Reports a 70 µs per-program dispatch floor and ~90 µs per-call latency for small fused programs.
- **Why it matters:** The 70 µs dispatch floor quantifies ANE scheduling granularity, setting the minimum viable power-inference sample interval for our IOReport-based ANE utilization proxy.

### Finding 3: First open end-to-end ANE training system — Orion
- **Source:** arXiv (Ramchand Kumaresan) — code at [mechramc/Orion](https://github.com/mechramc/Orion)
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 2026
- **Summary:** Orion bypasses CoreML via `_ANEClient` and `_ANECompiler` private APIs and provides the first open system for direct ANE execution and on-device training. The paper catalogs 20 previously undocumented restrictions on MIL IR programs, memory layout, and compilation limits.
- **Why it matters:** Most complete public documentation of `_ANEClient`/`_ANECompiler` API surface; the 20-restriction catalog informs what program shapes actually reach the ANE vs get silently rerouted to CPU/GPU.

### Finding 4: macOS 26 broke the ANE compile API — maderix Part 3 + maderix/ANE repo
- **Source:** maderix Substack — code at [maderix/ANE](https://github.com/maderix/ANE)
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** March 7, 2026
- **Summary:** Part 3 of the maderix M4 ANE series demonstrates full training (forward + backward + Adam) on Stories110M at 96 ms/step and 2.8W. Apple changed the ANE compile API in macOS 26, breaking the existing private framework access pattern; community contributor Steve Kromer patched it in PR #27 within days.
- **Why it matters:** Confirms the private ANE API surface is not stable across macOS major versions; monitorplus needs a version probe or compatibility shim for macOS 26.

### Finding 5: M1 confirmed to have two AMX blocks — "Above the Inner Loop"
- **Source:** arXiv (Deyvik Bhan, Georgia Institute of Technology)
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Demonstrates M1 has two distinct on-chip AMX blocks; a kernel using fine multi-thread panels to fill the second block plus pre-packed constant weights exceeds Accelerate fp32 GEMM by 1.17x on all twelve LLM prefill shapes tested. The inner loop is load-issue bound; speedup comes entirely from second-block occupancy and weight pre-packing.
- **Why it matters:** Dual AMX block microarchitecture is previously undocumented; if blocks are independently power-gated, IOReport energy deltas may distinguish single- vs dual-block AMX utilization.

### Finding 6: M3/M4 PMU has 16-bit event selectors and PMCR0 kernel preemption — clf3 blog
- **Source:** blog.clf3.org
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** 2026 (exact date not listed)
- **Summary:** Documents two critical M3/M4 PMU differences from M1/M2: event selector registers are 16-bit per slot (vs 8-bit on M1/M2), and `SYS_APL_PMCR0_EL1` is silently overwritten by a kernel background process every ~100 µs, causing user-space counter programming to lose its configuration within one tick.
- **Why it matters:** Both differences break a naïve M1-targeting kperf sidecar on M3/M4 — the privileged sidecar needs generation-conditional register access and a PMCR0 restoration loop.

### Finding 7: Apple M-series Linux PMU driver reaches v10 patchset — LKML Nick Chan
- **Source:** LKML
- **URL:** https://lkml.org/lkml/2026/1/1/91
- **Date:** January 1, 2026
- **Summary:** Nick Chan posted v10 of the Apple M-series PMU driver patchset for Linux (`drivers/perf: apple_m1: Support per-implementation PMU startup`). v10 indicates significant iteration; each revision encodes per-chip-generation PMU register semantics in navigable driver code.
- **Why it matters:** The per-implementation startup code is a public reference for M2/M3/M4 PMU register differences, complementing the clf3 blog findings above.

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
