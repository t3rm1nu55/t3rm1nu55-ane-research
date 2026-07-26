# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-26 — sweep (5 findings)

### Finding 1: Bryngelson — "Apple Neural Engine: Architecture, Programming, and Performance" (arXiv 2606.22283)
- **Source:** arXiv + GitHub [sbryngelson/ane-guide](https://github.com/sbryngelson/ane-guide) + [ane-guide.readthedocs.io](https://ane-guide.readthedocs.io/en/latest/)
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** 2026-06-21
- **Summary:** Reverse-engineered reference for the entire ANE stack — from the datapath and roofline through the below-CoreML dispatch route, compiler and on-disk program format, weight-compression scheme, kernel driver, firmware, and XPC command protocol — based on direct measurement on Apple silicon and static analysis of the private runtime. Covers all M-series chip generations across nine structured parts plus appendices.
- **Why it matters:** First public documentation of the ANE kernel driver and XPC command protocol — the layer a t3rm1nu55-monitorplus privileged sidecar would need to intercept for ANE busy/idle telemetry beyond IOReport energy sampling.

### Finding 2: Bryngelson — ANEForge: Python direct ANE dispatch without CoreML (arXiv 2606.17090)
- **Source:** arXiv + GitHub [sbryngelson/ANEForge](https://github.com/sbryngelson/ANEForge)
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** 2026-06-12
- **Summary:** Python package that compiles a lazy tensor graph (58 fused + 19 bridge operators) into a single ANE program and dispatches it through the ANE daemon and kernel-driver stack directly, bypassing CoreML. Achieves ~90 μs per call near the hardware's 70 μs dispatch floor; runs full inference and training on the ANE. Targets macOS 14+ on Apple Silicon.
- **Why it matters:** The private `_ANEClient`/`_ANECompiler` API surface is now fully characterized in working, compilable code — the clearest reference for a Rust FFI shim, and the 70 μs dispatch floor bounds any timing-based utilization proxy.

### Finding 3: Orion — First open end-to-end ANE training system (arXiv 2603.06728)
- **Source:** arXiv + GitHub [mechramc/Orion](https://github.com/mechramc/Orion)
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** 2026-03-16
- **Summary:** First open system combining direct ANE execution, a compiler pipeline, and stable multi-step training via `_ANEClient`/`_ANECompiler` private APIs with IOSurface-backed zero-copy tensor I/O. Catalogs 20 restrictions on ANE MIL IR programs. Achieves 170+ tokens/s on GPT-2 (M4 Max) and 8.5× recompilation speedup via delta compilation.
- **Why it matters:** The ANEDaemon XPC + IOSurface dispatch path is now documented in a compilable project — a concrete integration target for a privileged sidecar observing ANE activity.

### Finding 4: "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" (arXiv 2606.25426)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** 2026-06 (submitted June 2026)
- **Summary:** Detailed AMX microarchitectural analysis for M1–M3. The AMX inner loop is load-issue bound; single-thread throughput falls to 610–680 GFLOPS (under half the load-free rate) when operand loads interleave with the FMA32 stream. Documents two on-chip AMX blocks per M1 and multi-thread panel strategies that outperform Accelerate at all 12 tested LLM GEMM shapes.
- **Why it matters:** First public quantitative characterisation of the two-AMX-block topology and throughput ceiling — informs what events a hypothetical kperf AMX counter should expose (load-stall rate, per-block utilisation).

### Finding 5: LKML — Nick Chan v10: Linux Apple PMU driver extended to A7–A11 and T2 (Jan 2026)
- **Source:** LKML
- **URL:** https://lkml.org/lkml/2026/1/1/82
- **Date:** 2026-01-01 (v10; iterated from v8 in Oct 2025 through v9 Dec 2025)
- **Summary:** 21-patch series extending the Linux `drivers/perf/apple_m1` PMU driver to cover Apple A7–A11 mobile SoCs and the T2 chip. Adds per-implementation event tables, per-implementation counter counts, and per-implementation PMU startup. Complements the existing M1/M2 upstream PMU work.
- **Why it matters:** More Apple chip generations now have public per-implementation PMU event tables in upstream Linux — a useful cross-reference when mapping kperf event names across generations for the macOS sidecar.

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
