# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-15 — sweep (6 findings)

### Finding 1: Complete ANE reverse-engineering guide published (A11–M5 coverage)

- **Source:** arXiv:2606.22283 / ane-guide.readthedocs.io / github.com/sbryngelson/ane-guide
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 2026
- **Summary:** Spencer H. Bryngelson (Georgia Tech) published the most comprehensive public reverse-engineering of the ANE ever produced, covering A11 through A18 and M1 through M5. The guide documents the datapath and roofline, the dispatch route below CoreML via the private `aned` stack, the compiler and on-disk program format, weight compression, and — crucially — the kernel driver, firmware, and command protocol. Direct measurements were taken on M1 and M5 hardware.
- **Why it matters:** The kernel driver and command protocol documentation is the missing piece for understanding whether an ANE utilization hook exists below CoreML; this is the closest the field has come to a complete architectural spec.

### Finding 2: ANEForge — Python direct-ANE library with roofline benchmarking

- **Source:** arXiv:2606.17090 / github.com/sbryngelson/ANEForge
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 2026
- **Summary:** Companion work to Finding 1 by the same author — a Python library that compiles tensor graphs into single ANE programs and dispatches them directly via the same `aned` private stack used by CoreML/MPSGraph/Espresso, bypassing CoreML's inference-only constraint to enable training. Ships `bench/roofline_suite.py` which "fingerprints your machine and measures its numeric cliffs" and uses `powermetrics` for whole-package energy measurement.
- **Why it matters:** The roofline benchmark suite is a reference implementation for ANE throughput measurement; the e5rt dispatch shim may expose a path to utilization sampling that t3rm1nu55-monitorplus could model.

### Finding 3: maderix/ANE Part 3 published + SRAM probing code

- **Source:** maderix Substack / github.com/maderix/ANE
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** 2026 (post April)
- **Summary:** The maderix ANE series reached Part 3 (full backpropagation on the ANE without CoreML or GPU), and the companion GitHub repo is now live with benchmarking tools: `inmem_peak.m` (peak TFLOPS), `inmem_bench.m` (dispatch latency), `ane_int8_bench.m` (INT8 vs FP16 throughput), `sram_bench.m`, and `sram_probe.m` (SRAM size/layout exploration). Measured peak: 18.6 TOPS (FP16), 35.1 TOPS (INT8 W8A8) on M4. Utilization under real training workloads is reportedly 5–9% of peak.
- **Why it matters:** The `sram_probe.m` and `sram_bench.m` tools directly probe ANE internal memory layout using `_ANEClient`; this approach could inform an ANE busy-state inference method for the monitor.

### Finding 4: AMX microarchitecture characterized — M1 has two AMX blocks per core

- **Source:** arXiv:2606.25426
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Deyvik Bhan (Georgia Tech) published a direct-AMX GEMM kernel that beats Accelerate's cblas_sgemm by 1.17x at LLM prefill scales by exploiting the M1's two on-chip AMX blocks per core via fine multi-thread panels — the key finding being that the M1 AMX is load-issue bound (throughput falls to ~640 GFLOPS when any operand load interleaves with FMA32, against a ~1.4 TFLOPS load-free rate). The paper documents the two-AMX-block-per-core structure explicitly for M1–M3.
- **Why it matters:** Confirms M1–M3 have two independent AMX blocks per P-core; any AMX utilization counter approach must account for both blocks and the load-issue bottleneck that makes throughput a poor proxy for compute saturation.

### Finding 5: kperf/kpc PMU confirmed working on M4 Pro; L1D refill sector is 64 B not 128 B

- **Source:** arXiv:2606.27098
- **URL:** https://arxiv.org/abs/2606.27098
- **Date:** June 2026
- **Summary:** Faruk Alpay and Barış Başaran used kperf/kpc as root to program fixed and configurable PMU counters on an M4 Pro (fixed cycles/instructions, L1D load misses, L1D refills, L2-TLB misses, data table walks) to study GPU–CPU cache residual state. A hardware discrepancy was found: macOS reports 128-byte cache lines, but PMU data shows the L1D refill sector is 64 bytes.
- **Why it matters:** Validates that kperf/kpc PMU programming still works on M4 Pro under current macOS; the paper is a current working reference implementation for the exact FFI pattern t3rm1nu55-monitorplus's kperf sidecar uses.

### Finding 6: Orion — 20 ANE operation constraints catalogued (14 previously unknown)

- **Source:** arXiv:2603.06728
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 2026
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" extends the public ANE constraint catalogue to 20 restrictions (14 previously undocumented), and demonstrates stable multi-step LLM training directly on the ANE by combining direct execution with a new compiler pipeline. Predates the last sweep but was not captured in the initial journal seed.
- **Why it matters:** The constraint catalogue is reference material for understanding which operations the ANE can accept from a custom dispatch path — directly relevant to any attempt to issue probe workloads for utilization inference.

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
