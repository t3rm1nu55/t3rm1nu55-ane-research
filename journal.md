# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-09-01 — sweep (3 findings)

### Finding 1: Full ANE reverse-engineering reference — architecture, compiler, driver, firmware

- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 21, 2026
- **Summary:** Spencer Bryngelson's "Apple Neural Engine: Architecture, Programming, and Performance" is the most thorough public reverse-engineered account of the ANE yet. It documents the fp16 datapath (fp32-class accumulator), the roofline bound, the dispatch route below CoreML, the MIL compiler and on-disk program format, the weight-compression scheme, and the kernel driver, firmware, and command protocol. A companion reference guide is live at `sbryngelson/ane-guide` on GitHub.
- **Why it matters:** The kernel-driver and firmware documentation is the most likely place to discover whether the ANE exposes any hardware performance counter registers accessible from the host CPU; this should be the primary read for v2 ANE work.

### Finding 2: ANEForge — direct Python-to-ANE dispatch without CoreML

- **Source:** arXiv / GitHub / PyPI
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 12, 2026
- **Summary:** ANEForge (`sbryngelson/ANEForge`) compiles a lazy tensor graph (58 fused operators, 19 bridge operators) to a single ANE program dispatched through the same ANE daemon and kernel-driver stack Apple uses internally, bypassing CoreML entirely. It supports forward, backward, and Adam optimizer steps, ONNX import, LLM decode/prefill, and model compression. No hardware performance counters are exposed, but it is the most capable open direct-dispatch stack available.
- **Why it matters:** Provides a reference implementation of the full direct-ANE call path (_ANEClient → daemon → driver → firmware); any future counter probe for t3rm1nu55-monitorplus would need to follow this same dispatch chain.

### Finding 3: AMX inner-loop microbenchmark — M1 AMX is load-issue bound, not FMA-bound

- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 24, 2026
- **Summary:** Bhan's "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" characterizes M1 AMX throughput via microbenchmark: the inner loop is load-issue bound, with peak single-thread throughput collapsing to ~610–680 GFLOPS when operand loads interleave with the FMA32 stream (vs. the load-free theoretical rate). The speedup over Accelerate comes from finer multi-thread panel scheduling, not a faster inner loop. Uses algorithmic microbenchmarks, not hardware PMU counters.
- **Why it matters:** Establishes that AMX throughput can be meaningfully characterized from software timing alone without hardware counter access; narrows the "why AMX utilization matters" framing for v2 scope decisions.

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
