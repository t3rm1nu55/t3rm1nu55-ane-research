# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-28 — sweep (4 findings)

### Finding 1: Comprehensive ANE architecture reverse-engineering paper (arxiv:2606.22283)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 21, 2026
- **Summary:** Spencer Bryngelson (Georgia Tech) published a full reverse-engineered reference for the Apple Neural Engine covering the datapath, throughput rooflines, dispatch route below CoreML, the compiler and on-disk program format, weight-compression scheme, kernel driver, and firmware/command protocol. Companion web guide at https://ane-guide.readthedocs.io and code at https://github.com/sbryngelson/ane-guide.
- **Why it matters:** Most complete public documentation of the ANE's private API surface to date; the dispatch route section maps exactly the path needed to inject utilization hooks below CoreML for t3rm1nu55-monitorplus.

### Finding 2: ANEForge — direct ANE dispatch without CoreML (arxiv:2606.17090)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 12, 2026
- **Summary:** Companion Python package by the same author; compiles a lazy tensor graph (58 fused operators) into a single ANE program dispatched through the native daemon/driver stack without CoreML. A small fused call completes in ~90 µs, near the engine's documented 70 µs dispatch floor. Code at https://github.com/sbryngelson/ANEForge.
- **Why it matters:** The 70 µs dispatch-latency floor calibrates IOReport energy-delta sampling rates; the dispatch mechanism demonstrates feasibility of ANE active-state inference without hardware counters.

### Finding 3: kperf/PMU microbenchmarks characterize M1 AMX inner loop (arxiv:2606.25426)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Uses kperf PMU microbenchmarks to prove the M1 AMX inner loop is load-issue bound — any interleaved operand load drops throughput to 610–680 GFLOPS, under half the load-free rate. The 1.17× speedup over Accelerate's GEMM paths comes from multi-thread panel sizing and weight pre-packing, not a faster inner loop.
- **Why it matters:** Demonstrates that kperf can characterize AMX behavior indirectly via load/FMA ratio events — the closest published approach to AMX counter exposure using existing kperf infrastructure.

### Finding 4: NPUMoE — MoE inference scheduled to ANE, IOReport energy methodology (arxiv:2604.18788)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2604.18788
- **Date:** April 20, 2026
- **Summary:** Introduces NPUMoE, which offloads static MoE expert computation to the ANE, reporting 1.32×–5.55× latency reduction and 1.81×–7.37× energy improvement over CPU/GPU baselines on M-series. Energy measurement is IOReport-based; uses offline expert calibration to avoid dynamic shape problems.
- **Why it matters:** Validates the IOReport energy approach for ANE activity inference at scale; confirms the power-as-proxy-for-utilization model holds under heterogeneous LLM workloads.

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
