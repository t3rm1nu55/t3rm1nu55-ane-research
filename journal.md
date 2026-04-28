# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-04-28 — sweep (2 findings)

### Finding 1: macmon exposes itself as a Rust library crate in v0.7.1

- **Source:** vladkens/macmon (GitHub)
- **URL:** https://github.com/vladkens/macmon
- **Date:** April 15, 2026 (v0.7.1 release)
- **Summary:** macmon v0.7.1 adds `feat: expose macmon as a library crate`, making its IOReport access patterns available as a Cargo dependency rather than code that must be vendored. The same release adds an HTTP server mode with JSON and Prometheus endpoints and fixes CPU usage calculation on Ultra chips.
- **Why it matters:** t3rm1nu55-monitorplus currently vendors macmon's IOReport access pattern; switching to a direct crate dependency would reduce maintenance burden and automatically track upstream IOReport channel additions including any future ANE-specific channels.

### Finding 2: k06a/macpow — new untracked IOReport power tool with documented ANE channel naming

- **Source:** k06a/macpow (GitHub, previously untracked)
- **URL:** https://github.com/k06a/macpow
- **Date:** April 9, 2026 (v0.1.17 release)
- **Summary:** macpow is a real-time per-component power TUI (CPU E/P cores, GPU, ANE, DRAM, Media Engine) for M1–M5+. Its documentation explicitly names the IOReport Energy Model channel conventions: bare block name `ANE` on single-die chips and `DIE_N_ANE` on Ultra multi-die chips. It reads IOReport, SMC, IORegistry, and Mach kernel APIs.
- **Why it matters:** Cross-referencing its channel naming against macmon and socpowerbud may surface any ANE IOReport channels that our current implementation misses, and the Ultra-chip naming convention (`DIE_N_ANE`) is directly relevant if M-Ultra support is added to monitorplus.

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
