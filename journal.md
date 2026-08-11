# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-11 — sweep (3 findings)

### Finding 1: "Apple Neural Engine: Architecture, Programming, and Performance" (arXiv:2606.22283)
- **Source:** arXiv / sbryngelson/ane-guide (Spencer H. Bryngelson, Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.22283 — guide: https://ane-guide.readthedocs.io
- **Date:** June 21, 2026
- **Summary:** The most complete public reverse-engineering of the ANE stack to date, derived from direct measurement and static decompilation. Covers the full path from userspace dispatch to kernel driver, firmware, and the ANE command protocol — including the datapath roofline, the CoreML bypass route, the compiler and on-disk program format, and the weight-compression scheme. Each claim is labelled measured, decompile-derived, or predicted.
- **Why it matters:** The kernel driver and command protocol sections are the missing piece for any future attempt to expose ANE utilization counters from the host; this is now the primary reference.

### Finding 2: ANEForge — Python direct-ANE dispatch library (arXiv:2606.17090)
- **Source:** arXiv + GitHub (sbryngelson/ANEForge) + PyPI (`aneforge`)
- **URL:** https://arxiv.org/abs/2606.17090 — repo: https://github.com/sbryngelson/ANEForge
- **Date:** June 2026
- **Summary:** A Python package that compiles a lazy tensor graph (58 fused + 19 native operators) directly to ANE programs and dispatches them via the same daemon/kernel-driver stack Apple uses internally — no CoreML, no Metal. Supports forward pass, backward pass, and Adam optimizer on the ANE.
- **Why it matters:** Provides a working, open reference implementation of the ANE dispatch protocol and daemon interaction, which is the API surface most likely to surface utilization hooks.

### Finding 3: AMX dual-block characterization on M1 (arXiv:2606.25426)
- **Source:** arXiv (Deyvik Bhan, Georgia Institute of Technology)
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 24, 2026
- **Summary:** Detailed performance analysis of M1's Apple Matrix Extension (AMX), revealing that M1 contains two on-chip AMX blocks (one per P-cluster), that the inner GEMM loop is load-issue bound at 610–680 GFLOPS single-threaded, and that both blocks can be exploited via multi-threading. A hand-written direct-AMX SGEMM kernel exceeds Accelerate by 1.17×.
- **Why it matters:** First public confirmation of the dual-AMX-block topology on M1 and a measured throughput ceiling — essential baseline for designing any future AMX utilization metric.

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
