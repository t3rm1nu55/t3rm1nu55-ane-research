# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-27 — sweep (3 findings)

### Finding 1: macmon now exposes raw CPU + GPU active residency ratios
- **Source:** vladkens/macmon
- **URL:** https://github.com/vladkens/macmon/commit/3010f1fb7e
- **Date:** 2026-06-09
- **Summary:** macmon's `calc_freq()` function was updated to return a third value — `usage_ratio` — representing raw active residency (time in any non-idle P-state / total time), separate from the existing frequency-weighted `cpu_usage_pct`. New `Metrics` fields added: `cpu_active_ratio`, `ecpu_active_ratio`, `pcpu_active_ratio`, `ecpu_core_active_ratios`, `pcpu_core_active_ratios`, and `gpu_active_ratio`. All are derived from the existing IOReport "CPU Core Performance States" channel, not a new channel.
- **Why it matters:** t3rm1nu55-monitorplus should mirror these new derived IOReport metrics; `cpu_active_ratio` is a cleaner "was the CPU doing work?" signal uncontaminated by DVFS frequency scaling, and `gpu_active_ratio` fills the same role for the GPU.

### Finding 2: m1n1 copies ARM Activity Monitor counter-direction registers for M3 (AGTCNTRDIR)
- **Source:** AsahiLinux/m1n1
- **URL:** https://github.com/AsahiLinux/m1n1/commit/d16d7353da
- **Date:** 2026-05-29
- **Summary:** Asahi's hypervisor now copies `AGTCNTRDIR_EL1` and `AGTCNTRDIR_EL12` (ARM Activity Monitor Counter Direction registers) from the primary to secondary CPU cores on M3 devices flagged with the `counter_redirect` CPU feature. These are ARM AMU extension registers (FEAT_AMUv1p1) governing which EL counters are attributed to; macOS 14.8.3 writes them unconditionally on M3. This is the first m1n1 commit to explicitly handle Apple's AMU `counter_redirect` feature.
- **Why it matters:** Confirms M3 implements ARM's Activity Monitors Unit (AMU) extension, which provides fixed-function hardware counters (instruction retirements, constant cycles, memory stalls) potentially readable from EL0 without kperf/kpc. Investigating whether macOS gates EL0 AMU access via AMUSERENR_EL0 is a concrete new research direction.

### Finding 3: m1n1 begins T6040 chip bring-up (likely M4 Pro)
- **Source:** AsahiLinux/m1n1
- **URL:** https://github.com/AsahiLinux/m1n1/commit/e7061018d0
- **Date:** 2026-06-14
- **Summary:** Florian Klink adds initial T6040 SoC definitions to m1n1 — MIDR entries in `midr.h`, SoC ID constant in `soc.h`, and minimal CPU chicken-bit config in `chickens.c`. Following Apple's chip numbering convention (T6030=M3 Pro, T6031=M3 Max, T6034=M3 Ultra), T6040 is likely M4 Pro. This is the very first Asahi bring-up commit for this SoC.
- **Why it matters:** Each Asahi chip bring-up precedes register mapping and PMU event table work by weeks; tracking this from the start ensures we catch any M4 Pro PMU counter additions or `counter_redirect` register changes as they land.

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
