# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-04-24 — sweep (3 findings)

### Finding 1: Orion — first open system for direct ANE training/inference, documents compilation limits
- **Source:** arXiv 2603.06728 / GitHub mechramc/Orion
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** 2026-03-06
- **Summary:** Orion is the first open end-to-end runtime that bypasses CoreML entirely to drive the ANE directly via the private `_ANEClient` and `_ANECompiler` APIs. It documents a critical operational constraint: each process is limited to approximately 119 ANE model compilations before subsequent calls silently fail. On an M4 Max it achieves 170+ tokens/s (GPT-2 124M inference) and trains a 110M-parameter transformer in 22 minutes by achieving an 8.5× recompilation speedup.
- **Why it matters:** The ~119 compilation-per-process limit is a hard wall for any probe-based ANE utilization estimation approach; any monitoring strategy that injects synthetic workloads to measure ANE occupancy must account for this budget.

### Finding 2: maderix ANE Part 3 — training on ANE + SRAM bandwidth probing tools published
- **Source:** maderix Substack / GitHub maderix/ANE
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** 2026-03-07
- **Summary:** Part 3 of the maderix M4 ANE series demonstrates the first public full forward+backward pass on the ANE for transformer training (109M params). The companion `maderix/ANE` GitHub repo ships `sram_bench.m` for SRAM bandwidth probing and `ane_int8_bench.m`; measured peak is 35.1 TOPS INT8 / 18.6 TOPS FP16 (slightly revised from the 19 TFLOPS figure in the seed). No raw hardware counters are exposed; SRAM probing is black-box bandwidth inference.
- **Why it matters:** SRAM bandwidth probing is the most fine-grained sub-IOReport visibility into ANE activity currently published; confirms that IOReport energy deltas remain the only real-time telemetry path available without kernel access.

### Finding 3: XTC paper — documents KPep database as the mechanism for kperf event discovery on Apple Silicon
- **Source:** arXiv 2512.16512
- **URL:** https://arxiv.org/abs/2512.16512
- **Date:** 2025-12-18 (missed by initial seed)
- **Summary:** XTC is a cross-platform AI-workload operator benchmarking harness that uses Apple's undocumented KPerf system interface and the per-chip KPep plist databases (`/usr/share/kpep/a14.plist`, `a15.plist`, `as4.plist`, etc.) to read hardware performance counters on Apple Silicon. It confirms the KPep database approach as the canonical path for automated event discovery across chip generations.
- **Why it matters:** Confirms that scanning the KPep plist database is the right method for discovering any new AMX-labeled or ANE-adjacent counter events; no such events are present in the current databases, but the methodology is validated.

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
