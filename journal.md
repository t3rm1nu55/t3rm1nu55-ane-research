# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-23 — sweep (5 findings)

### Finding 1: Comprehensive ANE Architecture Reference — arXiv 2606.22283

- **Source:** arXiv / sbryngelson/ane-guide
- **URL:** https://arxiv.org/abs/2606.22283 / https://github.com/sbryngelson/ane-guide
- **Date:** June 2026
- **Summary:** "Apple Neural Engine: Architecture, Programming, and Performance" (Bryngelson) is the deepest public reverse-engineering of the ANE to date, covering the datapath and roofline, compiler and on-disk program format, weight-compression scheme, kernel driver, firmware, and command protocol — all derived from direct hardware measurement and static analysis of private frameworks. A companion web edition is at ane-guide.readthedocs.io; the reference is also used by ANEForge (Finding 2) and the Orion paper (Finding 3).
- **Why it matters:** Kernel driver and firmware protocol documentation is the closest public map to where ANE utilization counters would live. Reviewing the ane-guide for counter-related IOKit interfaces is now a concrete next step for t3rm1nu55-monitorplus.

### Finding 2: ANEForge — Direct ANE Dispatch via Python, PyPI-Available

- **Source:** arXiv 2606.17090 / sbryngelson/ANEForge
- **URL:** https://arxiv.org/abs/2606.17090 / https://github.com/sbryngelson/ANEForge
- **Date:** June 12, 2026
- **Summary:** ANEForge compiles a lazy tensor graph (58 fused operators + 19 native bridge operators) into a single ANE program and dispatches it through the same private `aned` daemon and IOKit kernel-driver path used by CoreML, MPSGraph, and Espresso — without CoreML. Training (forward pass, backward pass, Adam) is fully supported; the package is available on PyPI as `aneforge`.
- **Why it matters:** ANEForge is the lowest-friction public way to drive the ANE kernel driver directly. Pairing ANEForge dispatches with IOReport Energy Model sampling could yield per-dispatch energy deltas and makes the kernel driver path concrete enough to investigate for utilization counter IOKit properties.

### Finding 3: Orion — First Public LLM Training Directly on ANE

- **Source:** arXiv 2603.06728 / mechramc/Orion (GitHub)
- **URL:** https://arxiv.org/abs/2603.06728 / https://github.com/mechramc/Orion
- **Date:** March 6, 2026
- **Summary:** Orion is the first open end-to-end LLM training and inference runtime for the Apple Neural Engine, bypassing CoreML entirely via Apple's private `_ANEClient` and `_ANECompiler` APIs. The paper documents achieving 94% ANE utilization with deep operation graphs (16–64 ops) and demonstrates training a 109M-parameter transformer with checkpoint resume. It builds on the maderix foundational API work.
- **Why it matters:** The 94% ANE utilization claim requires a measurement basis — the methodology used to derive this figure likely reveals either a hardware counter or a power-inferred proxy that could directly inform t3rm1nu55-monitorplus's ANE telemetry design.

### Finding 4: AMX Two-Block Architecture Discovered via PMU Counters

- **Source:** arXiv 2606.25426
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 24, 2026
- **Summary:** "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" (Bhan) uses hardware PMU counters to characterize the M1 AMX and discovers that M1 contains two on-chip AMX blocks, not one as previously assumed. The AMX inner loop is load-issue bound at ~610–680 GFLOPS under mixed load/FMA workloads; exploiting fine multi-thread panels across both blocks exceeds Accelerate's BNNS Graph path by 1.17×.
- **Why it matters:** Two AMX blocks changes the utilization model — any future AMX counter must aggregate across both units. The PMU counter methodology used here (load-issue bound characterization) is a direct template for AMX event discovery work.

### Finding 5: kperf/kpc Counter Usage on Apple M4 Pro Documented

- **Source:** arXiv 2606.27098
- **URL:** https://arxiv.org/abs/2606.27098
- **Date:** June 25, 2026
- **Summary:** "Residual GPU Cache State on Apple M4 Pro" (Alpay & Başaran) programs kperf/kpc as root on an M4 Pro, using fixed (cycles, instructions) plus configurable events: retired L1D load misses, L1D refill sectors, L2-TLB data misses, and data table walks. The paper pairs kperf measurements with IOReport histogram data for hardware grounding.
- **Why it matters:** Confirms working kperf counter configurations on M4 Pro generation hardware and provides a concrete template for validating t3rm1nu55-monitorplus's privileged kperf sidecar counter setup on M4 Pro targets.

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
