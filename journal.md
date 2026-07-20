# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-20 — sweep (5 findings)

### Finding 1: Comprehensive ANE architecture paper with companion open-source repo
- **Source:** arXiv 2606.22283
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 27, 2026 (v2)
- **Summary:** Spencer Bryngelson (Georgia Tech) published a comprehensive reverse-engineered ANE architecture reference covering the full dispatch path below CoreML, on-disk program format, roofline characterization (M1 ANE: ~12 TFLOP/s fp16, ridge point ~141 FLOP/byte), weight-compression scheme, and the kernel driver/firmware protocol. Companion repo at github.com/sbryngelson/ane-guide and rendered docs at ane-guide.readthedocs.io.
- **Why it matters:** The most complete public map of the ANE stack to date; the dispatch-path documentation is the closest existing guide to where utilization hooks would need to be inserted.

### Finding 2: ANEForge — direct ANE dispatch library bypassing CoreML
- **Source:** arXiv 2606.17090
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 12, 2026
- **Summary:** Bryngelson also released ANEForge, a Python library that compiles 58 fused operators directly into ANE programs and dispatches them through the ANE daemon/kernel-driver stack, bypassing CoreML entirely. A small fused program completes in ~90 µs (near the 70 µs per-program dispatch floor); ResNet-18 runs in 0.33 ms.
- **Why it matters:** ANEForge operates at exactly the dispatch layer through which any real-time ANE utilization hook would need to run; its implementation is the most concrete public reference for programmatic ANE access.

### Finding 3: kperf/kpc counter methodology confirmed working on M4 Pro
- **Source:** arXiv 2606.27098
- **URL:** https://arxiv.org/abs/2606.27098
- **Date:** June 2026
- **Summary:** Faruk Alpay and Baris Basaran characterize residual GPU cache contamination on a 14-core M4 Pro by running the private kperf/kpc interface as root, counting 64-byte L1D refill sectors and fixing P-core L1D capacity at 128 KiB — the same privileged counter access pattern monitorplus already uses.
- **Why it matters:** Confirms our privileged-sidecar kperf approach works on M4 Pro silicon; the cache-event counter methodology used is directly transferable to future monitorplus counter work.

### Finding 4: AMX inner loop characterised as load-issue bound on M1–M3
- **Source:** arXiv 2606.25426
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Deyvik Bhan (Georgia Tech) studied single-precision GEMM on the Apple AMX coprocessor across M1–M3, determining the inner loop is load-issue bound rather than compute-bound; achieved 1.17× over all three Accelerate fp32 paths via fine multi-thread panel sizing and pre-packed constant weights.
- **Why it matters:** The load-issue-bound finding means L1D/L2 cache-refill kperf events correlate with AMX activity — a candidate indirect AMX utilization proxy worth investigating in monitorplus.

### Finding 5: AMX single-core fp32 peak confirmed at ~350 GFLOPS on M1
- **Source:** arXiv 2605.05699
- **URL:** https://arxiv.org/abs/2605.05699
- **Date:** May 2026
- **Summary:** A fused Metal kernel study reports reaching 45% of a "documented 350 GFLOPS single-core AMX peak" at batch size 1024, giving the most precise publicly cited figure for M1 AMX fp32 throughput ceiling to date.
- **Why it matters:** Provides a calibration ceiling for any inference-based AMX utilization estimate in monitorplus — knowing the hardware peak lets a cache-delta proxy be expressed as a percentage.

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
