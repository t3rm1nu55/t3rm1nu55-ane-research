# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-05 — sweep (3 findings)

All three findings come from a coordinated Georgia Tech computational-physics group burst of preprints in June 2026. Together they represent the deepest public treatment of ANE/AMX internals to date.

### Finding 1: Comprehensive ANE Reverse Engineering — Architecture, Kernel Driver & Firmware Protocol

- **Source:** arXiv preprint 2606.22283
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 21, 2026
- **Summary:** Spencer Bryngelson (Georgia Tech) published the most comprehensive public reverse-engineering of the Apple Neural Engine to date, covering chip generations A11 through M5. The paper documents the ANE datapath, throughput/energy roofline, full dispatch route below CoreML, compiler and on-disk program format, weight-compression scheme, and — critically — the kernel driver, firmware, and command protocol beneath them. Direct measurements were taken on M1 and M5 hardware.
- **Why it matters:** The documented kernel-driver + firmware command protocol is the most promising path yet to determining whether ANE performance counters exist and are IOKit-accessible — directly relevant to t3rm1nu55-monitorplus ANE telemetry.

### Finding 2: ANEForge — Open-Source Python Direct ANE Dispatch Without CoreML

- **Source:** arXiv preprint 2606.17090
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 2026
- **Summary:** The same Georgia Tech group released ANEForge, a Python package that compiles a lazy tensor graph (58 fused operators, 19 native bridge operators) into ANE programs dispatched directly via the ANE daemon and kernel-driver stack, bypassing CoreML entirely. It supports both inference and training (forward pass, backward pass, Adam optimizer update) on hardware Apple designed for inference only.
- **Why it matters:** ANEForge is an open-source reference implementation of the direct ANE dispatch path. Examining its instrumentation approach may surface undocumented counter or timing signals within the ANE kernel driver.

### Finding 3: M1 AMX Has Two Blocks Per Chip; M4+ Replaced AMX With ARM SME

- **Source:** arXiv preprint 2606.25426
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Deyvik Bhan (Georgia Tech) characterised the M1 AMX via microbenchmark and found two key facts: (1) the M1 chip contains **two AMX blocks** (not one, as was widely assumed), and the AMX inner loop is load-issue bound at roughly 610–680 GFLOPS single-thread; (2) the M4 and later chips replaced AMX entirely with ARM's Scalable Matrix Extension (SME). Throughput gains over Accelerate come from filling the second AMX block via fine multi-threading, not from a faster inner loop.
- **Why it matters:** The two-block architecture changes any thread model for AMX monitoring on M1–M3; the M4+ SME transition means t3rm1nu55-monitorplus will need separate monitoring logic per chip generation for matrix-coprocessor activity.

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
