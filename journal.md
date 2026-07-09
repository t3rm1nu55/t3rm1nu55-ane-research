# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-09 — sweep (5 findings)

### Finding 1: "Apple Neural Engine: Architecture, Programming, and Performance" (Bryngelson, arXiv:2606.22283)
- **Source:** arXiv / Georgia Tech
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 2026
- **Summary:** The deepest public reverse-engineered reference for the ANE to date, covering the full vertical stack: datapath roofline, dispatch route below CoreML, compiler and on-disk program format, weight-compression scheme, kernel driver, firmware, and command protocol. Spans A11–A18 and M1–M5 with per-chip target tables. Companion repo at https://github.com/sbryngelson/ane-guide.
- **Why it matters:** Directly answers "what is below CoreML" — the kernel driver and firmware interface are now publicly documented, providing a map for any future counter-extraction attempt on ANE.

### Finding 2: ANEForge — Python for direct ANE computation (arXiv:2606.17090)
- **Source:** arXiv / Georgia Tech (same author as Finding 1)
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 12, 2026
- **Summary:** A Python package that compiles a lazy tensor graph (58 fused operators + 19 native bridge operators) into a single native ANE program, bypassing CoreML entirely. Supports int8/int4/sparse weights, keeps state resident across steps, and runs forward + backward + optimizer updates on ANE under macOS 14+.
- **Why it matters:** The first open package to reach ANE without CoreML — establishes that the direct-access path (previously only demonstrated by maderix) is reproducible and documented.

### Finding 3: PMU event counter register layout on M3 and M4 (blog.clf3.org)
- **Source:** ClF3's blog
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** ~June 2026 (surfaced with Asahi Linux 7.1 report)
- **Summary:** Documents the generation gap in Apple PMU ESR register layout: M3/M4 use 64-bit PMESR regs with 16 bits per event slot (vs 8 bits on M1/M2), and performance counters are 64-bit with bit 63 triggering PMI. This makes M1/M2 counter enumeration code non-portable to M3/M4 without a chip-generation branch. Referenced by the Linux kernel Nick Chan [PATCH v10] at https://lkml.org/lkml/2026/1/1/88.
- **Why it matters:** Any kperf sidecar that hardcodes M1-era ESR layout will silently misread counters on M3/M4 hardware — a direct correctness bug for t3rm1nu55-monitorplus users on MacBook Pro M3/M4.

### Finding 4: "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" (arXiv:2606.25426)
- **Source:** arXiv / Georgia Institute of Technology
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Characterizes the M1 AMX inner loop as load-issue bound — when any operand load interleaves with the FMA32 stream, single-thread throughput collapses to ~610–680 GFLOPS (under half of load-free peak). Identifies that Accelerate underuses the M1's second on-chip AMX block for K≥N shapes. A direct-AMX kernel beats Accelerate by 1.17× across 12 LLM prefill shapes.
- **Why it matters:** First public characterization of AMX block topology and throughput ceilings; the load-issue bottleneck is the kind of insight that would need to be reflected in any AMX utilization metric (a 50% throughput collapse is not visible from energy alone).

### Finding 5: "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" (arXiv:2603.06728)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 2026 (missed in initial seed)
- **Summary:** Demonstrates that deep operation graphs (16–64 ops) achieve 94% ANE utilization and runs full LLM training (forward, backward, Adam update) on ANE. Companion GitHub implementation at https://github.com/mechramc/Orion.
- **Why it matters:** The 94% utilization figure is measured via black-box benchmarking (wall-clock / TOPS), not a hardware counter — confirms the gap this repo is tracking and establishes the benchmark baseline for when counter-derived utilization eventually becomes available.

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
