# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-09 — sweep (5 findings)

### Finding 1: Full ANE architecture reference paper (arXiv 2606.22283)
- **Source:** arXiv / sbryngelson/ane-guide
- **URL:** https://arxiv.org/abs/2606.22283 · https://ane-guide.readthedocs.io · https://github.com/sbryngelson/ane-guide
- **Date:** June 21, 2026 (v2: June 27, 2026)
- **Summary:** Spencer Bryngelson (Georgia Tech) published the first complete reverse-engineered account of ANE hardware: datapath, roofline, dispatch route below CoreML, MIL compiler, on-disk program format, weight-compression scheme, kernel driver, firmware, and command protocol. Based on direct measurement and static analysis of the private runtime and kernel extension. Companion web edition at ane-guide.readthedocs.io with full GitHub source.
- **Why it matters:** This is the reference document the field has been missing — it defines what internal ANE resources exist and therefore what a future counter could plausibly track.

### Finding 2: ANEForge — direct ANE Python package (arXiv 2606.17090)
- **Source:** arXiv / sbryngelson/ANEForge / PyPI
- **URL:** https://arxiv.org/abs/2606.17090 · https://github.com/sbryngelson/ANEForge · https://pypi.org/project/aneforge/
- **Date:** June 12, 2026
- **Summary:** Companion to finding 1. ANEForge is a Python package (published to PyPI) that compiles a lazy tensor graph (58 fused operators, 19 native bridge operators) into a single ANE program and dispatches it via the same ANE daemon/kernel-driver stack as Apple's internal framework — without CoreML. Supports full training: forward, backward, and Adam update all compile to ANE programs. Ensures dispatch goes to ANE only (no silent CPU/GPU fallback).
- **Why it matters:** Operationalizes direct `_ANEClient`/`_ANECompiler` access with a clean API; the dispatch-path tracing in the source code is directly applicable to building ANE utilization inference in t3rm1nu55-monitorplus.

### Finding 3: AMX microarchitectural paper — two on-chip blocks, load-issue bounds (arXiv 2606.25426)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 24, 2026
- **Summary:** Deyvik Bhan (Georgia Tech) published the first microarchitecture-depth study of M1 AMX, revealing: (1) M1 has two on-chip AMX blocks that can be filled with fine multi-thread panels, (2) the inner loop is load-issue bound — any operand load interleaving with FMA32 drops throughput to ~610–680 GFLOPS (under half load-free rate), (3) M4 and later switched from Apple AMX to ARM SME (Scalable Matrix Extension). A direct-AMX kernel achieves 1.58× geomean over BNNSMatMul across 12 LLM prefill GEMMs.
- **Why it matters:** M4+ uses SME not AMX — this splits the counter problem by chip generation. The paper's characterization of AMX block topology explains why existing kperf event lists show no AMX-specific events (it is not on the PMU bus).

### Finding 4: maderix/ANE — open-source ANE training via private APIs, with benchmarks
- **Source:** GitHub
- **URL:** https://github.com/maderix/ANE
- **Date:** Active through mid-2026 (42 commits; companion to Substack Part 3, March 2026)
- **Summary:** Open-source implementation of forward/backward transformer training directly on ANE via `_ANEClient`, `_ANECompiler`, and `_ANEInMemoryModelDescriptor`. Includes GPU↔ANE zero-copy pipelines and INT8 W8A8 quantization. Benchmarks on M4: FP16 18.6 TOPS (128×conv, 512ch), INT8 W8A8 35.1 TOPS (1.88× speedup). Sustained utilization measured at only 5–9% of peak, highlighting dispatch overhead as the dominant bottleneck.
- **Why it matters:** The benchmark harness in this repo is the most credible public measurement of ANE throughput and the 5–9% utilization ceiling establishes what a hypothetical counter would actually report.

### Finding 5: jiegec/apple-pmu adds M5 (H17G Hidra) counter definitions
- **Source:** GitHub — jiegec/apple-pmu
- **URL:** https://github.com/jiegec/apple-pmu · https://github.com/jiegec/apple-pmu/blob/master/as5.md
- **Date:** 2026 (M5 shipped mid-2026)
- **Summary:** The jiegec/apple-pmu repo, which dumps PMU counter definitions from `/usr/share/kpep`, has been extended with M5 (H17G Hidra) support: files as5.md, as5-1.md, as5-2.md. M5 adds LD_SRC_* (load data-source tracking), PL2 cache events, and SME (Scalable Matrix Extension) engine counters not present in M4. The ESR register format already changed at M3/M4 (8-bit → 16-bit per event); M5 continues the 16-bit layout.
- **Why it matters:** Confirms no AMX/ANE-specific named events in M5 kpep either, and documents the SME engine counter namespace — the path to watch for any future Apple-disclosed accelerator events.

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
