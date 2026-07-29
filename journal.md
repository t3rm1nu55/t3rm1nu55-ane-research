# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-29 — sweep (4 findings)

### Finding 1: "Apple Neural Engine: Architecture, Programming, and Performance" (arXiv:2606.22283)
- **Source:** arXiv / sbryngelson/ane-guide
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 2026
- **Summary:** Spencer H. Bryngelson (Georgia Tech) published the most comprehensive public reverse-engineering of the ANE to date: datapath, throughput roofline, dispatch route below CoreML, compiler, on-disk program format, weight compression, kernel driver, firmware, and command protocol — derived from direct measurement on Apple Silicon and static analysis of private frameworks. Companion GitHub repo and docs: https://github.com/sbryngelson/ane-guide and https://ane-guide.readthedocs.io.
- **Why it matters:** The kernel driver and command protocol documentation is the layer at which ANE hardware counters (if any are exposed to the host) would live; this is the most actionable new reference for any future counter discovery attempt.

### Finding 2: "ANEForge: Python for direct computation on the Apple Neural Engine" (arXiv:2606.17090)
- **Source:** arXiv / sbryngelson/ANEForge
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 12, 2026
- **Summary:** Companion paper to Finding 1. ANEForge (GitHub: https://github.com/sbryngelson/ANEForge) is a Python package that compiles a lazy tensor graph of 58 fused operators directly into ANE programs, bypassing CoreML, with measured dispatch latency ~90 µs approaching the hardware dispatch floor of ~70 µs. Supports inference, forward/backward pass, and optimizer steps on ANE.
- **Why it matters:** Establishes the ~70 µs per-program ANE dispatch floor, which sets a hard bound on any sampling-based utilization metric; the direct-dispatch pathway it uses is the same layer where performance counter reads would occur.

### Finding 3: "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" (arXiv:2606.25426)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Hand-coded AMX programming for LLM prefill GEMM that exceeds Apple Accelerate throughput on M1, achieved by directly programming the undocumented AMX coprocessor instructions. The paper characterizes AMX throughput at the instruction level for matrix-heavy LLM workloads.
- **Why it matters:** Establishes the AMX instruction-level throughput model that is a prerequisite for identifying which kperf events (if any) track AMX activity; the measurement methodology is the current best proxy for AMX counter design.

### Finding 4: maderix/ANE — transformer training via reverse-engineered private ANE APIs
- **Source:** GitHub
- **URL:** https://github.com/maderix/ANE
- **Date:** June 2026
- **Summary:** New GitHub repo from maderix (author of the previously-tracked M4 ANE Substack series) implementing full transformer training on ANE via private `_ANEClient`/`_ANECompiler`/`_ANEInMemoryModelDescriptor` APIs. Includes `sram_bench.m` (SRAM bandwidth probing) and `inmem_peak.m` (peak TFLOPS via 2048×2048 matmul); measurement is timing-based, not counter-based. Reports 18.6 TFLOPS FP16 and 35.1 TFLOPS INT8 on M4 ANE.
- **Why it matters:** `sram_bench.m`'s timing-based SRAM bandwidth probing is the current best available proxy for ANE activity; the pattern is directly adaptable for a "utilization via bandwidth probe" approach in the t3rm1nu55-monitorplus Rust sidecar.

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
