# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-07 — sweep (5 findings)

### Finding 1: Comprehensive ANE reverse engineering published — datapath, driver, firmware, command protocol

- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 21, 2026
- **Summary:** Spencer Bryngelson (Georgia Tech) published a full reverse-engineered account of the Apple Neural Engine based on direct hardware measurement and static analysis of the private runtime, CoreML compiler, kernel driver, and firmware. Documents the ANE datapath and roofline, the dispatch route that reaches the ANE *below* CoreML, the on-disk program format, weight-compression scheme, and the kernel driver/firmware/command protocol in detail. This is the most complete public reference on ANE internals to date.
- **Why it matters:** The dispatch route and kernel driver protocol described here are exactly what t3rm1nu55-monitorplus would need to hook into for ANE utilization telemetry; supersedes all prior partial accounts.

### Finding 2: ANEForge — Python package for direct ANE computation without CoreML

- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 12, 2026
- **Summary:** Companion tool to Finding 1, also from Bryngelson's group at Georgia Tech. ANEForge compiles a lazy tensor graph (58 fused operators, 19 native bridge operators) into a single ANE program and dispatches it through the ANE daemon and kernel driver directly — bypassing CoreML entirely. Works under macOS 14+. Enables reproducible ANE benchmarking without CoreML's opaque scheduling.
- **Why it matters:** Provides a working reference implementation of the ANE dispatch path described in 2606.22283; a Rust port of this dispatch approach is the clearest path to ANE utilization tracking in t3rm1nu55-monitorplus.

### Finding 3: M1 AMX inner loop characterized using hardware PMU counters

- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Deyvik Bhan (Georgia Tech) uses hardware PMU counters to determine that the M1 AMX inner loop is *load-issue bound*, running at 610–680 GFLOPS when any operand load is interleaved with the FMA32 stream — well below the load-free rate. Demonstrates that Accelerate can be beaten at fp32 GEMM (1.17× over BNNS Graph) using panel tiling and weight pre-packing rather than a faster kernel. Geometric mean 1.58× over BNNSMatMul.
- **Why it matters:** Confirms that PMU counters *can* be used to characterize AMX behavior from the host CPU; the counter groups used to isolate the load-issue bound are a reference for what kperf events to target for AMX activity inference.

### Finding 4: macmon adds IOReport ANE active residency ratio channel

- **Source:** vladkens/macmon (GitHub)
- **URL:** https://github.com/vladkens/macmon/commit/3010f1f
- **Date:** June 9, 2026
- **Summary:** macmon added active residency ratio metrics (PR #61), exposing the IOReport channel that reports the fraction of time CPU clusters, GPU, and ANE are in active state over each sample interval. This is distinct from the energy-joule channel already tracked — it is a *duty-cycle* metric rather than a power metric.
- **Why it matters:** t3rm1nu55-monitorplus should add this IOReport channel; it provides a coarse "ANE utilization %" figure via IOReport without requiring any private API access.

### Finding 5: m1n1 expands PMP power-management tracing and adds M3 (T8122) cpufreq support

- **Source:** AsahiLinux/m1n1 (GitHub)
- **URL:** https://github.com/AsahiLinux/m1n1/commits/main
- **Date:** June–July 2026
- **Summary:** m1n1 landed two batches of relevant changes: (1) PMP (Power Management Processor) tracing was expanded to cover more Apple Silicon device variants, deepening the Asahi team's understanding of the Apple power telemetry path; (2) cpufreq support for M3 (T8122) was added, along with a batch of Apple-proprietary `SYS_IMP_APL_HID*` chicken-bit registers moved to the common register set. A commit copying `AGTCNTRDIR*` registers for M3 hypervisor initialization may be relevant to counter-direction register handling.
- **Why it matters:** The M3 cpufreq support extends the power-state model to the generation preceding M4; the PMP tracing work is the Asahi team's primary upstream handle on Apple's power management telemetry that we otherwise see only through IOReport.

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
