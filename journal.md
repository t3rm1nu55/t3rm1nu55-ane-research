# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-26 — sweep (5 findings)

### Finding 1: maderix Part 3 — First Transformer Training on ANE + M5 Confirmation
- **Source:** maderix Substack
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** March 2026
- **Summary:** Part 3 trains a 109M-parameter transformer on the M4 ANE (full forward + backward pass, Adam optimizer) and confirms the M5 ANE is the same H16G die family as M4, with the same weight-baking constraint. IOReportLegend reveals the ANE has independent adaptive clocking and multiple hardware/software power triggers not previously documented. A single transformer layer benchmark (dim=768, seq=512) measures 11.2% ANE utilization (1.78 TFLOPS of 15.8 theoretical peak).
- **Why it matters:** The 11.2% utilization figure is the first concrete runtime utilization measurement from an ANE workload; the IOReportLegend adaptive-clock channels are new leads for richer power telemetry in monitorplus.

### Finding 2: maderix/ANE — Open-Source Direct ANE Access Code
- **Source:** GitHub (maderix/ANE)
- **URL:** https://github.com/maderix/ANE
- **Date:** March 2026
- **Summary:** Companion code repo for the maderix series: a working implementation of direct ANE access via reverse-engineered `_ANEClient` and `_ANECompiler` private APIs, bypassing CoreML entirely. Maps 40+ private IOKit classes to the kernel driver and implements in-memory model compilation and dispatch.
- **Why it matters:** First public, working code exercising the full IOKit ANE stack — directly inspectable for any counter or utilization surface the driver exposes at the IOKit boundary.

### Finding 3: Orion (arXiv:2603.06728) — Systematic ANE Characterization Paper
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** 2026-03-06
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" extends the maderix reverse-engineering work into a systematic catalog of 20 ANE compilation restrictions (14 newly discovered MIL IR, memory, and I/O constraints). Deep operation graphs (16–64 ops) achieve 94% ANE utilization; the system is fully open-source at github.com/mechramc/Orion.
- **Why it matters:** Establishes 94% as a measured utilization ceiling achievable via the private API stack and maps the constraint space — useful context for deciding what workload patterns produce high-utilization IOReport readings.

### Finding 4: Nick Chan LKML v10 — Linux PMU Driver Documents M3/M4 Event-Encoding Break
- **Source:** LKML
- **URL:** https://lkml.org/lkml/2026/1/1/82
- **Date:** 2026-01-01
- **Summary:** A 21-patch set (`[PATCH v10 00/21] drivers/perf: apple_m1: Add Apple A7-A11, T2 SoC support`) refactors the Linux Apple PMU driver around per-implementation event tables and counter counts. A key technical fact surfaced: M3/M4 PMU ESR registers are 64-bit with 16-bit event encoding per slot (vs. 8-bit on M1/M2), and performance counters are 64-bit with bit 63 as PMI trigger (vs. bit 47 on M1).
- **Why it matters:** The 16-bit event encoding on M3/M4 is a mandatory correction for any kperf FFI targeting those chips — all existing reference implementations (ibireme gist, bugsiki analysis) use M1/M2's 8-bit event fields and will misfire silently on M3/M4.

### Finding 5: clf3.org — PMU Event Counters on Apple M3 and M4 (new untracked source)
- **Source:** blog.clf3.org
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** Unknown (appeared post-April 2026)
- **Summary:** Independently confirms M3/M4 PMU differences (64-bit counters, 16-bit ESR event encoding) and documents a critical runtime constraint: writes to `SYS_APL_PMCR0_EL1` without kernel patching are overwritten by a kernel process within ~100 μs, making userspace-only PMU enablement infeasible for sustained counter reads on unpatched systems.
- **Why it matters:** The ~100 μs PMCR0 clobber window validates the necessity of the privileged sidecar architecture in monitorplus — the sidecar must hold kernel entitlements or a patched PMCR0, not just write the register once per sample.

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
