# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-12 — sweep (6 findings)

### Finding 1: Bryngelson — full ANE hardware architecture reference (arXiv:2606.22283)
- **Source:** arXiv / Georgia Tech
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 21, 2026
- **Summary:** Spencer Bryngelson published a reverse-engineered hardware reference for the Apple Neural Engine covering A11–A18 and M1–M5, built from direct measurement and static analysis of the private runtime, compiler, kernel driver, and firmware. Documents the full datapath and roofline, dispatch path below CoreML to the ANE daemon and kernel driver, on-disk program format, weight-compression scheme, and command protocol. Companion web edition at ane-guide.readthedocs.io; source at github.com/sbryngelson/ane-guide.
- **Why it matters:** The ANE dispatch path and command protocol are now publicly documented for M1–M5; this is the prerequisite reference for any ANE counter or utilization instrumentation work in t3rm1nu55-monitorplus.

### Finding 2: ANEForge — Python library for direct ANE dispatch, bypassing CoreML (arXiv:2606.17090)
- **Source:** arXiv / Georgia Tech
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 12, 2026
- **Summary:** ANEForge (Bryngelson) is a Python package that compiles lazy tensor graphs (58 fused + 19 bridge operators) into a single ANE program dispatched directly through the ANE daemon and kernel-driver stack, bypassing CoreML entirely; supports forward/backward passes and Adam optimization. GitHub: github.com/sbryngelson/ANEForge.
- **Why it matters:** Bypassing CoreML's routing uncertainty enables deterministic, controlled ANE dispatch; the kernel-driver path it uses is the same path any future ANE counter or power instrumentation would traverse — it is now inspectable in Python.

### Finding 3: AMX inner-loop microarchitecture — load-issue bound on M1–M3, SME on M4+ (arXiv:2606.25426)
- **Source:** arXiv / Georgia Tech
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 24, 2026
- **Summary:** Bhan systematically benchmarks the M1–M3 AMX coprocessor and SME on M4+ via microbenchmarks, finding the M1 AMX inner loop is load-issue bound: any operand load interleaved with the FMA32 stream drops single-thread throughput to ~610–680 GFLOPS, under half the load-free peak. Speedup over Accelerate comes from fine multi-thread panel tiling (exploiting M1's second on-chip AMX block) and pre-packing constant weights, not a faster inner loop.
- **Why it matters:** Confirms no publicly extractable AMX hardware counter exists for utilization; throughput characterization requires microbenchmark proxies, and the load-issue bound model is the current best public understanding of AMX performance on M1–M3.

### Finding 4: macmon Frida research confirms powermetrics ANE Energy Model subscription (macmon 40f4e46)
- **Source:** vladkens/macmon
- **URL:** https://github.com/vladkens/macmon/commit/40f4e46
- **Date:** July 23, 2026
- **Summary:** macmon added Frida-hook scripts for `IOReportCreateSamples`, `IOReportCreateSamplesDelta`, and `IOReportCreateSubscription` to reverse-engineer powermetrics' subscription strategy; found powermetrics opens exactly 3 IOReport subscriptions per cycle, with the Energy Model subscription explicitly covering 136 channels including CPU, GPU, ANE, DRAM, display, media, and SRAM.
- **Why it matters:** Confirms the IOReport Energy Model path is the correct channel for ANE power sampling and documents the full subscription scope (136 channels) that t3rm1nu55-monitorplus's IOReport subscriber should request.

### Finding 5: macmon documents M5 IOReport three-tier cluster channel naming (macmon 6919d77)
- **Source:** vladkens/macmon
- **URL:** https://github.com/vladkens/macmon/commit/6919d77
- **Date:** August 4, 2026
- **Summary:** macmon's M3 Ultra per-core metrics fix added explicit M5 IOReport channel documentation: M5 uses a three-tier cluster layout — ECPU (Efficiency), MCPU (mid-tier Performance), PCPU (Super) — vs. M1–M4's two-tier ECPU/PCPU; Ultra dies use `DIE_N_*` prefix, e.g. `DIE_0_PCPU1_CPU0`. CPU core key types widened to String to handle cross-generation naming collisions.
- **Why it matters:** M5 IOReport channel prefix mapping is now documented upstream in macmon; t3rm1nu55-monitorplus will need this when adding M5 support to avoid misidentifying MCPU cluster channels.

### Finding 6: Asahi m1n1 — M5 PMGR support, M3 cpufreq, and M3 AGTCNTRDIR counter register (m1n1)
- **Source:** AsahiLinux/m1n1
- **URL:** https://github.com/AsahiLinux/m1n1/commit/fcaf476
- **Date:** May–June 2026
- **Summary:** m1n1 added PMGR power-manager device support for M4 Pro/Max, A18 Pro, and M5 (fcaf476, May 15); M3 (T8122) and M3 Pro (T6030/t6034) variants gained working cpufreq support (June 2026); and M3 secondary cores received a copy of the `AGTCNTRDIR` (Activity Monitor Generator Counter Direction) register (d16d735, June 5).
- **Why it matters:** The AGTCNTRDIR register copy for M3 secondaries suggests Asahi is wiring M3-specific activity counter direction state into the PMU bringup path — a signal that M3 Linux PMU support is incrementally advancing even without a formal LKML patchset.

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
