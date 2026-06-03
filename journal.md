# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-03 — sweep (4 findings)

### Finding 1: Orion — First Open End-to-End Framework for Direct ANE Programming
- **Source:** arXiv:2603.06728
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026 (published before last sweep; missed by initial seed)
- **Summary:** First open system that bypasses CoreML entirely via Apple's private `_ANEClient` and `_ANECompiler` APIs. Catalogs 20 constraints on ANE MIL IR programs, 14 of which are previously undocumented (including a 119-compilation-per-process hard limit that causes silent failures). Provides detailed hardware characterization of the M4 Max ANE: 38 TOPS INT8 across 16 cores, fp16-optimized, with measured throughput and latency. Achieves a 3.8× training speedup by patching weight files in-place instead of invoking `_ANECompiler` on each step.
- **Why it matters:** The deepest public characterization of ANE internals to date; the `_ANEClient` API surface documented here is the hook point for any future ANE utilization counter exposure in t3rm1nu55-monitorplus.

### Finding 2: maderix Part 3 + `maderix/ANE` GitHub repo — Working `_ANEClient` Implementation
- **Source:** maderix Substack Part 3; GitHub `maderix/ANE`
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b · https://github.com/maderix/ANE
- **Date:** March 7, 2026 (Substack Part 3); repo last commit March 10, 2026
- **Summary:** Completes the maderix M4 ANE series (Parts 1–3) with a from-scratch transformer training implementation running entirely on the ANE. The GitHub repo provides Objective-C source code that directly uses `_ANEClient`, `_ANECompiler`, and `_ANEInMemoryModelDescriptor`, demonstrating full forward pass, backward pass, gradient computation, and Adam optimizer updates on 109M parameters on hardware designed solely for inference. The references table previously tracked only Part 2.
- **Why it matters:** First public, open-source Objective-C reference implementation of direct `_ANEClient` access; the code is a working template for any attempt to build ANE dispatch monitoring or utilization hooks.

### Finding 3: LKML Nick Chan v10 — `apple_m1_cpu_pmu` Extended to Per-Implementation Event Tables
- **Source:** LKML (Linux Kernel Mailing List)
- **URL:** https://lkml.org/lkml/2026/1/1/82
- **Date:** January 1, 2026 (published before last sweep; missed by initial seed)
- **Summary:** 21-patch v10 series by Nick Chan extending the Linux `apple_m1_cpu_pmu` driver to support Apple A7–A11 SoCs. Key structural additions: per-implementation PMU event tables, per-implementation counter counts, 32-bit EL0 counter support, and per-implementation PMU startup sequences. This refactors the driver from M1-monolithic to chip-generic, requiring each implementation to declare its own event set.
- **Why it matters:** The per-implementation event tables reveal which kperf events each chip generation exposes; cross-referencing the Linux driver's event databases with macOS `/usr/share/kpep/` plists is the most reliable way to audit M2–M5 event coverage gaps for our kperf FFI.

### Finding 4: blog.clf3.org — M3/M4 PMU ESR Event-Slot Width Change Documented
- **Source:** ClF3's blog
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** Unknown (2025 or early 2026; untracked source)
- **Summary:** Documents a previously unreported architectural difference in M3/M4 PMU event selection registers: each event slot is 16 bits wide on M3/M4, versus 8 bits on M1/M2. This changes the bit-packing layout of the ESR (Event Selection Register) read and written by kperf, and means M3/M4 kperf configuration code cannot be shared with M1/M2 without a chip-generation branch.
- **Why it matters:** Our kperf FFI sidecar must handle both register formats; this is currently the only public documentation of this M3/M4 ESR width change and must inform the counter-configuration path in `t3rm1nu55-monitorplus`.

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
