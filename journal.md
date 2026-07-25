# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-25 — sweep (5 findings)

### Finding 1: "Apple Neural Engine: Architecture, Programming, and Performance" (arXiv:2606.22283)
- **Source:** arXiv + sbryngelson/ane-guide (GitHub)
- **URL:** https://arxiv.org/abs/2606.22283 · web edition: https://ane-guide.readthedocs.io
- **Date:** June 21, 2026
- **Summary:** Comprehensive reverse-engineered reference for the full ANE stack by Spencer H. Bryngelson (Georgia Tech). Documents the datapath, roofline, dispatch route below CoreML, compiler and on-disk program format, weight-compression, kernel driver, firmware, and command protocol. Every claim is flagged as measured, decompile-derived, or predicted. Companion GitHub repo at sbryngelson/ane-guide.
- **Why it matters:** The kernel driver + firmware + command protocol chapters are the first public documentation of the register-level ANE dispatch path — the exact layer where a future utilization counter would have to live.

### Finding 2: "ANEForge: Python for direct computation on the Apple Neural Engine" (arXiv:2606.17090)
- **Source:** arXiv + sbryngelson/ANEForge (GitHub)
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 2026
- **Summary:** Python library by the same author as ane-guide that compiles a tensor graph into one ANE program and dispatches it through the private `aned` stack (bypassing CoreML), the same path used by MPSGraph and Espresso. Companion repo sbryngelson/ANEForge.
- **Why it matters:** The `aned` dispatch path it exposes is instrumentable; counting dispatch events through this hook is a practical proxy utilization signal pending hardware-counter discovery.

### Finding 3: "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" (arXiv:2603.06728)
- **Source:** arXiv / GitHub: mechramc/Orion
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026 (predates last sweep but was not captured)
- **Summary:** First open end-to-end system for ANE LLM training and inference bypassing CoreML entirely via `_ANEClient` and `_ANECompiler` private APIs. Catalogs 20 ANE restrictions on MIL IR programs, memory layout, and compilation limits. Demonstrates GPT-2 124M inference (170+ tok/s) and Stories110M training.
- **Why it matters:** The constraint catalog and stable direct-API access method together establish the most complete public picture of the ANE's programming model; feeds directly into understanding what a counter-level hook would need to tolerate.

### Finding 4: "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" (arXiv:2606.25426)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Characterizes the M1 AMX at microarchitectural depth: finds that M1 contains two onchip AMX blocks (a second block that Accelerate systematically underuses) and that the inner loop is load-issue bound at ~610–680 GFLOPS vs ~1.4 TFLOPS load-free. A hand-written kernel exploiting both blocks with weight pre-packing beats Accelerate by 1.17× on LLM prefill GEMMs.
- **Why it matters:** First confirmed evidence of a dual-block AMX architecture on M1; any future kperf event scan for AMX should look for events distinguishing the two blocks, and the measured throughput values give calibration targets for inference-based utilization estimation.

### Finding 5: maderix Substack Part 3 — "Inside the M4 Apple Neural Engine, Part 3: Training"
- **Source:** maderix Substack
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** ~March 2026
- **Summary:** Third installment of the maderix ANE series (Parts 1–2 were already tracked), demonstrating backpropagation through ANE's fixed-function ops via the `_ANEClient`/`_ANECompiler` private API surface. Confirms the API is stable enough for multi-step training runs.
- **Why it matters:** Extends the maderix benchmarking baseline from inference into training, validating that the private API surface is robust enough for long-running measurement workloads.

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
