# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-13 — sweep (2 findings)

### Finding 1: "Apple Neural Engine: Architecture, Programming, and Performance" (arXiv:2606.22283)
- **Source:** arXiv (via tracked search string `"Apple Neural Engine" AND ("counter" OR "utilization" OR "benchmark")`)
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 2026
- **Summary:** Spencer Bryngelson (Georgia Tech) publishes the most comprehensive public reverse-engineering of the ANE to date, covering the datapath, throughput roofline, dispatch route below CoreML, compiler and on-disk program format, weight-compression scheme, and — critically — the **kernel driver, firmware, and command protocol** beneath them. Covers A11 through A18 and M1 through M5, with per-chip target tables and an operation-by-device matrix. Measurements taken directly on M1 and M5. Companion reference at [ane-guide.readthedocs.io](https://ane-guide.readthedocs.io) / [sbryngelson/ane-guide](https://github.com/sbryngelson/ane-guide).
- **Why it matters:** Kernel driver and command protocol documentation is the missing piece for constructing a privileged-sidecar path to the ANE in t3rm1nu55-monitorplus; this is the first public source that documents it at this depth.

### Finding 2: "ANEForge: Python for direct computation on the Apple Neural Engine" (arXiv:2606.17090)
- **Source:** arXiv (via tracked search string `"Apple Neural Engine" AND ("counter" OR "utilization" OR "benchmark")`)
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 12, 2026
- **Summary:** Same Georgia Tech group releases ANEForge, a Python package that compiles a lazy tensor graph (58 fused ops + 19 bridge ops) and dispatches it to the ANE through the undocumented `_ANEClient`/`_ANECompiler`/`aned` private stack — no CoreML. Supports cross-compilation for 28 ANE targets (M1–M5) from one machine, training (forward + backward + Adam), int8/int4/sparse weights, and latency estimation without running hardware. Verified on M5 Pro and M1 Max. GitHub: [sbryngelson/ANEForge](https://github.com/sbryngelson/ANEForge).
- **Why it matters:** Demonstrates that the private `_ANEClient` API surface is stable across all shipping M-series chips — a prerequisite for any library or sidecar that dispatches to or monitors the ANE. The latency-estimation without hardware capability hints at internal counters or roofline models worth investigating.

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
