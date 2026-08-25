# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-25 — sweep (4 findings)

### Finding 1: Comprehensive ANE Architectural Reference (M1–M5) — Bryngelson / Georgia Tech

- **Source:** arXiv 2606.22283 + GitHub [sbryngelson/ane-guide](https://github.com/sbryngelson/ane-guide)
- **URL:** https://arxiv.org/abs/2606.22283 / https://ane-guide.readthedocs.io
- **Date:** June 2026
- **Summary:** Spencer Bryngelson (Georgia Tech) published a reverse-engineered reference covering A11–A18 and M1–M5 ANE families, including the datapath and roofline, dispatch route below CoreML, compiler and on-disk program format, and kernel driver/firmware/command protocol — all derived from direct measurement and static analysis of the private runtime. Measurements are on M1 and M5. A companion reference manual site and GitHub repo are freely available.
- **Why it matters:** Most complete public ANE architectural document to date, now covering M5; directly informs any future ANE power-inference or utilization-estimation work in t3rm1nu55-monitorplus.

### Finding 2: ANEForge — Python direct ANE programming via private APIs

- **Source:** arXiv 2606.17090 + GitHub [sbryngelson/ANEForge](https://github.com/sbryngelson/ANEForge)
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 12, 2026
- **Summary:** ANEForge (also by Bryngelson) is an open-source Python package that compiles a tensor graph of 58 fused operators and dispatches the resulting ANE program through `_ANEClient`/`_ANECompiler`/`aned` — entirely bypassing CoreML. It supports both inference and on-device training, and is available on PyPI (`pip install aneforge`).
- **Why it matters:** Provides an up-to-date, tested symbol inventory for direct ANE access; the dispatch pattern is portable to Rust and could underpin a real-time ANE-load probe if hardware counters remain inaccessible.

### Finding 3: M4+ drops AMX for ARM SME — and as4/as5 kperf plists include SME counters

- **Source:** arXiv 2606.25426 + [jiegec/apple-pmu](https://github.com/jiegec/apple-pmu)
- **URL:** https://arxiv.org/abs/2606.25426 / https://github.com/jiegec/apple-pmu/blob/master/as4.md
- **Date:** June 24, 2026 (paper); as4/as5 kperf data extracted from macOS 15–26
- **Summary:** The paper "Above the Inner Loop" confirms M4 was the first public device with ARM Scalable Matrix Extension (SME), replacing Apple's proprietary AMX on M1–M3. The M1 AMX inner loop is load-issue bound at 610–680 GFLOPS single-thread. Separately, [jiegec/apple-pmu](https://github.com/jiegec/apple-pmu) now contains extracted kperf event plists for M4 (`as4`) and M5 (`as5`): `as4` adds ARM architectural events and **SME engine counters**; `as5` adds LD_SRC_* load data source events and PL2 cache events.
- **Why it matters:** The AMX→SME transition means t3rm1nu55-monitorplus must handle M4+ separately; the newly documented SME kperf events in `as4` are candidate counters to expose for matrix-unit utilization on M4/M5.

### Finding 4: SiliconScope — IOReport channel layout changed on macOS 26 for M4 Max / M5 Max

- **Source:** GitHub [kennss/SiliconScope](https://github.com/kennss/SiliconScope)
- **URL:** https://github.com/kennss/SiliconScope
- **Date:** 2026 (active repo, recent v2.4.0 release)
- **Summary:** SiliconScope is a new open-source sudoless SwiftUI monitor exposing ANE, Media Engine, and memory-bandwidth metrics from IOReport. A recent release fixed a regression where memory-bandwidth and Media Engine channels returned 0 on M4 Max and M5 Max under macOS 26 due to a changed IOReport layout. The fix involved updating the private IOReport channel names for those chip variants.
- **Why it matters:** Direct signal that Apple renamed IOReport channels between macOS 15 and macOS 26 on M4 Max/M5 Max — t3rm1nu55-monitorplus uses the same IOReport interface and is at risk of reading zeros on these chips without a parallel fix; SiliconScope's diff is the reference patch.

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
