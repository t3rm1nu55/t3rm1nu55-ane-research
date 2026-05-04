# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-04 — sweep (4 findings)

### Finding 1: M3/M4 PMU event encoding is 16-bit per slot (vs 8-bit on M1/M2)
- **Source:** ClF3's blog — "Utilizing PMU Event Counters on Apple M3 and M4"
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** ~2026 (exact date unknown)
- **Summary:** On M1/M2 the PMU `ESR` register encodes each event in 8 bits; on M3/M4 the register widens to 64 bits and each event slot is 16 bits. The post walks through `SYS_APL_PMCR0_EL1` and `SYS_APL_PMCR1_EL1` register semantics and confirms the fixed-counter count (2) and configurable-counter count (8) remain the same across generations.
- **Why it matters:** Any kperf sidecar that hard-codes M1/M2 event-encoding bit widths will silently produce wrong counter values on M3/M4 — this is a concrete porting requirement for the privileged sidecar.

### Finding 2: Orion — first open-source end-to-end ANE training system; 14 new MIL constraints catalogued
- **Source:** arXiv 2603.06728; GitHub mechramc/Orion
- **URL:** https://arxiv.org/abs/2603.06728 · https://github.com/mechramc/Orion
- **Date:** March 2026
- **Summary:** Orion bypasses CoreML entirely via `_ANEClient` and `_ANECompiler` private symbols, compiles a 27-op graph IR through five optimization passes to ANE-native MIL, and is the first system to demonstrate stable multi-step training with checkpoint resume directly on the ANE. The paper consolidates a catalog of 20 ANE operational constraints, 14 of which are newly discovered (including the ~119-compilation-per-process limit before silent failures begin).
- **Why it matters:** The constraint catalog is the most comprehensive public documentation of what the ANE compiler will and will not accept; the open-source implementation provides a reference access path (`_ANEClient` symbol table, IOSurface I/O pattern) that a future utilization probe could follow.

### Finding 3: mperf — portable kperf CLI with cross-generation event aliases for M1–M4
- **Source:** lambdafoo.com — "Quick Hardware Performance Counters on macOS ARM64"
- **URL:** https://lambdafoo.com/posts/2026-03-25-mperf-hardware-counters-macos.html
- **Date:** March 25, 2026
- **Summary:** `mperf` is a `perf-stat`-like CLI built on the ibireme kperf reverse-engineering gist that introduces portable aliases (`cycles`, `instructions`, `branch-misses`, `l1d-cache-misses`, etc.) that resolve to the correct chip-specific kpep event name at runtime; the tool confirmed stability from M1 through M4 without code changes.
- **Why it matters:** Demonstrates the correct abstraction pattern for a multi-generation kperf FFI layer; the alias-resolution approach is directly applicable to the privileged sidecar's event-configuration API.

### Finding 4: lauka — Apple Silicon PMU counter benchmark CLI with statistical aggregation
- **Source:** GitHub verte-zerg/lauka
- **URL:** https://github.com/verte-zerg/lauka
- **Date:** ~2026 (exact date unknown)
- **Summary:** `lauka` merges the `poop` and `scoop` kperf libraries into a CLI that records named PMU counters for a command under test and reports mean/stddev/min/max plus a delta column vs a baseline command; supports `lauka counters` to enumerate all available kpep events with compatibility flags.
- **Why it matters:** Another clean kperf reference implementation; the compatibility-flag enumeration subcommand is a useful pattern for discovering which events are valid on the host chip generation without hard-coding event IDs.

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
