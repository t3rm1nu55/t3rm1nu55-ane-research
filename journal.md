# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-23 — sweep (2 findings)

### Finding 1: macmon separates active residency ratios from frequency-blended utilization
- **Source:** vladkens/macmon
- **URL:** https://github.com/vladkens/macmon/issues/61
- **Date:** 2026-06-09
- **Summary:** PR #61 corrects macmon's CPU/GPU cluster utilization calculation. The old formula blended active residency with frequency scaling (`active_residency / total_residency * avg_freq / max_freq`), causing thermally-throttled but fully-busy clusters to appear partially idle. The fix surfaces raw active residency ratios directly from IOReport's "CPU Complex Performance States" channel as a metric independent of frequency.
- **Why it matters:** macmon is the IOReport reference for t3rm1nu55-monitorplus; if our project inherited the blended formula, cluster busy-fraction readings will be systematically wrong under thermal throttle — opened issue on main project to audit.

### Finding 2: Asahi m1n1 adds M3 (T8122) cpufreq support, confirms AMX throttle register layout
- **Source:** AsahiLinux/m1n1
- **URL:** https://github.com/AsahiLinux/m1n1/commit/59dd457
- **Date:** 2026-06-23
- **Summary:** Commit 59dd457 extends m1n1's cpufreq driver to the M3 SoC (T8122), confirming its DVFS register layout is identical to M2 Pro/Max (T6030/T6031). The `t8122_features` register set includes an AMX throttle register at cluster offset `0x40250`, consistent with M2 Pro/Max.
- **Why it matters:** Low priority — confirms M3 DVFS register compatibility and that the AMX throttle address `0x40250` applies unchanged to M3; useful baseline if future work reverse-engineers AMX activity state from register polling.

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
