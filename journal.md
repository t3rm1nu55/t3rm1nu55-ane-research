# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-06 — sweep (4 findings)

### Finding 1: Bryngelson 2026 — Comprehensive ANE reverse engineering, kernel driver and firmware
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 21, 2026
- **Summary:** Spencer Bryngelson published the first comprehensive reverse-engineered account of the Apple Neural Engine covering the datapath, dispatch route below CoreML, compiler and on-disk program format, weight-compression scheme, and critically the **kernel driver, firmware, and command protocol**. Based on direct measurement on Apple Silicon and static analysis of the private runtime, compiler, and kernel driver.
- **Why it matters:** Kernel driver and command protocol documentation is the most actionable ANE material ever published — may enable probing ANE utilization state directly without CoreML. Read before implementing any ANE counter path in the main project.

### Finding 2: ANEForge — Python library for direct ANE dispatch, exposing the raw dispatch path
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 12, 2026
- **Summary:** ANEForge (Bryngelson et al.) compiles a lazy tensor graph (58 fused operators, 19 native bridge operators) into a single ANE program and dispatches it directly below CoreML, with ~90 µs per-call overhead near the 70 µs hardware dispatch floor. Targets macOS 14+ on Apple Silicon. Companion to Finding 1.
- **Why it matters:** The raw dispatch path exposed by ANEForge could be instrumented to measure real ANE utilization; the 70 µs dispatch floor sets the minimum meaningful sample interval for utilization estimation.

### Finding 3: SiliconScope v3.1.4 — per-process ANE memory inspection, new IOReport channel map
- **Source:** github.com/kennss/SiliconScope
- **URL:** https://github.com/kennss/SiliconScope/releases
- **Date:** July 5, 2026
- **Summary:** SiliconScope v3.0 introduced per-process ANE memory inspection (sudoless) via a Process Inspector pane; v3.1.4 shipped July 5, 2026. The repo includes `docs/ioreport-channels.md` documenting ANE0/ANE1 energy channels and memory-bandwidth channels (including Media Engine and per-client DCS channels) not catalogued in other open-source tools.
- **Why it matters:** The channel map is a diff-able reference against the main project's IOReport subscriptions; the per-process ANE memory metric may correspond to an IOReport channel not yet subscribed by macmon or t3rm1nu55-monitorplus.

### Finding 4: Above the Inner Loop — M1 AMX load-issue bound characterization
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Microbenchmarks show the M1 AMX inner loop is load-issue bound at ~610–680 GFLOPS once any operand load interleaves with the FMA32 stream — under half the uncontended rate. Beating Accelerate requires multi-thread panel sizing and weight pre-packing, not a faster FMA loop.
- **Why it matters:** Establishes that peak AMX throughput is a poor proxy for effective utilization; any future AMX hardware-counter or energy-proxy metric must account for this load-issue bottleneck to be interpretable.

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
