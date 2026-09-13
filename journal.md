# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-09-13 — sweep (6 findings)

### Finding 1: SME engine counters now in M4 kpep — first PMU-visible matrix-engine events on Apple Silicon
- **Source:** jiegec/apple-pmu (GitHub)
- **URL:** https://github.com/jiegec/apple-pmu/blob/master/as4.md
- **Date:** Documented from M4 (as4.plist) kpep database; M4 shipped Oct 2024, M5 Ultra Aug 2026
- **Summary:** The M4's kpep PMU event database exposes `INST_SME_ENGINE_ALU` (0x8a3), `INST_SME_ENGINE_LD` (0x8a1), `INST_SME_ENGINE_ST` (0x8a2), `INST_SME_ENGINE_SCALARFP` (0x8a0), and `INST_SME_ENGINE_PACKING_FUSED` (0x529) — all accessible through the standard kperf/kpc counter interface. SME (ARM Scalable Matrix Extension) is Apple's publicly-named successor to AMX on M4+, and these events are the first hardware-counter-level visibility into matrix engine activity ever documented in kpep. The jiegec/apple-pmu repo also confirms as5 (M5) adds load data source tracking and PL2 cache events.
- **Why it matters:** Directly contradicts the "no AMX-specific PMU events" finding from the seed state; on M4+ hardware t3rm1nu55-monitorplus can program INST_SME_ENGINE_ALU via kperf to get a real matrix-engine instruction count.

### Finding 2: ANE Architecture, Programming, and Performance — first comprehensive reverse-engineered reference
- **Source:** arXiv (Spencer H. Bryngelson, Georgia Tech); companion repo sbryngelson/ane-guide
- **URL:** https://arxiv.org/abs/2606.22283 · https://github.com/sbryngelson/ane-guide · https://ane-guide.readthedocs.io
- **Date:** June 21, 2026
- **Summary:** Reverse-engineered reference covering the ANE datapath and roofline, dispatch route below CoreML, compiler and on-disk program format (E5 binary), weight-compression scheme, and the kernel driver / firmware / command protocol. Based on direct measurement on Apple Silicon and static analysis of AppleNeuralEngine.framework. This is the most thorough public ANE technical reference ever published, and it is open source.
- **Why it matters:** The driver/firmware chapter may reveal command-protocol hooks usable as dispatch counters; the datapath roofline establishes the theoretical ceiling for any ANE utilization metric.

### Finding 3: ANEForge — open Python framework for direct ANE computation without CoreML
- **Source:** arXiv (Spencer H. Bryngelson); GitHub sbryngelson/ANEForge; PyPI aneforge
- **URL:** https://arxiv.org/abs/2606.17090 · https://github.com/sbryngelson/ANEForge
- **Date:** June 12, 2026
- **Summary:** Python package compiling a lazy tensor graph of 58 fused operators into a single ANE program dispatched through the private `aned` stack, bypassing CoreML entirely. A small fused program completes in ~90μs, near the engine's measured 70μs per-program dispatch floor. Supports inference, forward/backward training, and keeps decoder/optimizer state resident across steps.
- **Why it matters:** First open tool exposing direct ANE dispatch timing; the per-program dispatch floor means dispatch-count × 70μs is a lower-bound utilization proxy even without hardware counters.

### Finding 4: "What actually runs" — paper uses ANE memory-controller byte counters as utilization signal
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2608.22110
- **Date:** August 22, 2026
- **Summary:** Measurement study of LLM placement and decode speed on the ANE. Authors "read the ANE's memory-controller byte counters during inference" to establish what actually ran vs. what the compiler intended. Finds placement is a function of how a computation is expressed: a fused RMSNorm is fully ANE-eligible while its arithmetically identical decomposition is CPU-only; int8/2-bit quantized models run at ~83% ANE residency vs. zero for an equivalent fp16 conv-heavy model.
- **Why it matters:** Confirms that ANE memory-controller byte counters are a real, readable utilization signal distinguishing ANE from CPU/GPU activity — a candidate metric for the main project beyond IOReport energy sampling.

### Finding 5: maderix — Part 3 of M4 ANE series published (training on ANE)
- **Source:** maderix Substack
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** March 2026
- **Summary:** Third installment of the tracked maderix ANE series. Covers full forward pass, backward pass, gradient computation, and Adam optimizer updates running on the M4 ANE via private `_ANEClient`/`_ANECompiler` APIs, scaling to Qwen3-0.6B (596M parameters). Completes the three-part series that was the seed reference for this repo.
- **Why it matters:** Confirms continuous bidirectional ANE dispatch from a single process is feasible; any monitoring solution must account for mixed inference/training workloads sharing the engine.

### Finding 6: LKML — New patch adds macOS 27 M2 SIMD event to Linux apple_m1 PMU driver
- **Source:** linux-perf-users mailing list (Myria Sarvay)
- **URL:** https://ratatoskr.run/linux-perf-users/2026/09/17557909/t
- **Date:** September 12, 2026
- **Summary:** Patch adds `INST_SIMD_ALU_VEC` (retired non-load/store vector SIMD instructions, counter 7 only) to the Linux `drivers/perf/apple_m1` driver, extracted from the macOS 27 M2 kpep database. Event is also settable on M1 CPUs. Demonstrates ongoing community effort to keep the Linux PMU driver synchronized with macOS kpep discoveries.
- **Why it matters:** Signals that macOS 27 kpep databases have new events worth scanning; the pattern of per-generation kpep additions is the same path through which SME engine counters (Finding 1) were discovered.

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
