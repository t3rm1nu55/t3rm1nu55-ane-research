# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-30 — sweep (3 findings)

### Finding 1: Comprehensive reverse-engineered ANE architecture guide (A11–M5)
- **Source:** arXiv 2606.22283 / sbryngelson/ane-guide
- **URL:** https://arxiv.org/abs/2606.22283 · https://github.com/sbryngelson/ane-guide · https://ane-guide.readthedocs.io
- **Date:** June 2026
- **Summary:** Spencer H. Bryngelson (Georgia Tech) published the most complete public reverse-engineering of the ANE to date: datapath, roofline bounds, dispatch route from CoreML to kernel driver to firmware, on-disk program format, weight-compression scheme, and command protocol — covering A11 through A18 and M1 through M5 with direct measurements on M1 and M5. Produced from static analysis of the private runtime, compiler, kernel driver, and firmware.
- **Why it matters:** First public map of the full ANE software/hardware stack; the kernel driver and firmware documentation is the most likely place to find counter-register offsets if they exist on any chip generation.

### Finding 2: ANEForge — Python toolkit for direct ANE computation without CoreML
- **Source:** arXiv 2606.17090 / sbryngelson/ANEForge
- **URL:** https://arxiv.org/abs/2606.17090 · https://github.com/sbryngelson/ANEForge
- **Date:** June 2026
- **Summary:** Companion paper to 2606.22283 by the same author. ANEForge is a Python package that compiles lazy tensor graphs (58 fused operators + 19 native bridge operators) directly to ANE programs dispatched through the ANE daemon and kernel driver, bypassing CoreML entirely. Supports both forward and backward passes, int8/int4/sparse weights, and fused attention — the first public end-to-end direct-ANE training toolkit.
- **Why it matters:** Demonstrates the full direct-ANE dispatch stack in open-source form; if any ANE utilization feedback surfaces at the daemon or driver layer, this codebase is where it would first appear.

### Finding 3: Working kperf/kpc counter methodology on M4 Pro documented
- **Source:** arXiv 2606.27098
- **URL:** https://arxiv.org/abs/2606.27098
- **Date:** June 2026
- **Summary:** Alpay & Başaran (2606.27098) program fixed and configurable PMU counters per-thread on an M4 Pro using the kperf/kpc private interface (fixed cycles/instructions, retired L1D load misses, L1D refills, L2-TLB data misses, data table walks); also uses IOReport histograms to disaggregate P-core, E-core, and AGX demand from live workloads. Primary focus is GPU cache state security, but the kperf methodology section is a validated reference implementation for M4 Pro.
- **Why it matters:** Confirms kperf counter access works on M4 Pro with the same approach as M1/M2; their IOReport + kperf combined methodology is directly applicable to the monitorplus kperf sidecar when adding M4 Pro support.

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
