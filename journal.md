# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-14 — sweep (2 findings)

### Finding 1: exelban/stats ships working ANE utilization via IOReport time-in-state
- **Source:** exelban/stats (https://github.com/exelban/stats)
- **URL:** https://github.com/exelban/stats/commit/abbdfc1
- **Date:** April 9, 2026 (shipped in v2.12.9)
- **Summary:** exelban/stats v2.12.9 added ANE utilization percentage to its GPU module by querying the `"SoC Stats"`/`"Cluster Power States"` IOReport group, filtering for `ANE`-prefixed channel names, and computing `Σ(delta_active) / Σ(delta_total)` via `IOReportStateGetResidency`. This is a direct time-in-state utilization ratio — not power-derived — and is the first confirmed open-source use of this IOReport channel group for ANE activity tracking. The implementation is ARM64-only and sudoless.
- **Why it matters:** The exact IOReport group+channel pattern (`SoC Stats / Cluster Power States / ANE*`) is directly reusable in t3rm1nu55-monitorplus as a Rust `IOReport` FFI call, unlocking a real utilization percentage rather than a power-inference proxy.

### Finding 2: exelban/stats publishes calibrated max-ANE-power constants per chip generation
- **Source:** exelban/stats (https://github.com/exelban/stats)
- **URL:** https://github.com/exelban/stats/commit/685c7ab
- **Date:** April 24, 2026
- **Summary:** A follow-up commit revised ANE utilization to use the `"Energy Model"` IOReport channel, normalising against per-chip max-power constants: M1 2.0 W, M4 6.0 W, M5 8.0 W. While the power-normalisation path is less precise than time-in-state, the M5 constant (8.0 W) is the first publicly calibrated figure for the M5 ANE, superseding the maderix M4 measurement (2.8 W peak) that was the previous best reference.
- **Why it matters:** Provides a fallback normalization formula and an actionable M5 max-power constant for any power-derived ANE utilization path in t3rm1nu55-monitorplus.

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
