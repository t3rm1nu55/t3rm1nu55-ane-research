# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-28 — sweep (3 findings)

### Finding 1: Comprehensive ANE architecture paper and guide (arXiv 2606.22283)
- **Source:** arXiv / sbryngelson/ane-guide (GitHub + ReadTheDocs)
- **URL:** https://arxiv.org/abs/2606.22283 · https://github.com/sbryngelson/ane-guide · https://ane-guide.readthedocs.io
- **Date:** June 21, 2026 (v2 revision June 27, 2026)
- **Summary:** Spencer Bryngelson (Georgia Tech) published the first systematic reverse-engineered account of the Apple Neural Engine, covering M1–M5 and A11–A18 families. The paper documents the ANE datapath, roofline performance bounds, compiler and on-disk program format, weight-compression scheme, and the kernel driver, firmware, and command protocol below CoreML. Per-chip target tables and an operation-by-device matrix are included; direct measurements are on M1 and M5.
- **Why it matters:** The kernel driver and command protocol documentation is the deepest public map of the ANE's software stack — a prerequisite for any future attempt to hook utilization counters or power telemetry below the CoreML abstraction.

### Finding 2: ANEForge — open Python library for direct ANE access (arXiv 2606.17090)
- **Source:** arXiv / sbryngelson/ANEForge (GitHub + PyPI)
- **URL:** https://arxiv.org/abs/2606.17090 · https://github.com/sbryngelson/ANEForge
- **Date:** June 12, 2026 (paper); June 23, 2026 (tool launch)
- **Summary:** Bryngelson also released ANEForge, a pip-installable Python package that compiles a lazy tensor graph (58 fused operators, 19 native bridge operators) directly into ANE programs without CoreML, using the private `_ANEClient` and `_ANECompiler` APIs. Training workloads (forward + backward + Adam) run fully on the ANE.
- **Why it matters:** First openly distributed library for direct ANE access; demonstrates the `_ANEClient` private API surface is stable enough to ship as a dependency. The dispatch path it exposes is exactly the hook point where power or timing measurements could be injected to infer utilization.

### Finding 3: kperf/kpc usage confirmed on M4 Pro hardware (arXiv 2606.27098)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.27098
- **Date:** June 25, 2026
- **Summary:** Faruk Alpay and Baris Basaran used the kperf/kpc private macOS interface to measure L1D refill sectors and distinguish cache-line conflict behavior on a 14-core Apple M4 Pro. The methodology cross-validates PMU hardware counters against IOReport histograms and STREAM/BabelStream benchmarks.
- **Why it matters:** Confirms kperf is functional and usable on M4 Pro as of macOS ~15.x; the cross-validation methodology against IOReport is directly applicable to how t3rm1nu55-monitorplus validates its own PMU readings.

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
