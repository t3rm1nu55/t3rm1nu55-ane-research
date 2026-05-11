# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-11 — sweep (3 findings)

### Finding 1: AsahiLinux/m1n1 gains initial T8140 (A18/A18 Pro) support
- **Source:** AsahiLinux/m1n1
- **URL:** https://github.com/AsahiLinux/m1n1/commit/0adfe2b
- **Date:** May 6, 2026
- **Summary:** Asahi's m1n1 bootloader adds preliminary recognition of T8140 (Apple A18/A18 Pro), the chip used in iPhone 16 Pro and the MacBook Neo — Apple's first A-series-based Mac (released March 2026). The commit adds UART base address, MIDR P/E-core identifiers, and SMP initialization, inheriting M4-generation feature flags.
- **Why it matters:** T8140/A18 Pro is now in Mac hardware; as Asahi deepens this chip's RE, PMU register documentation will follow and directly apply to monitoring the MacBook Neo.

### Finding 2: macmon v0.7.2 fixes IOReport frequency-scaling channels for MacBook Neo
- **Source:** vladkens/macmon
- **URL:** https://github.com/vladkens/macmon/releases
- **Date:** May 2, 2026
- **Summary:** macmon v0.7.2 fixes CPU frequency display (issue #57) on MacBook Neo (A18 Pro), confirming the A18 Pro uses different IOReport channel layout or frequency-scaling semantics compared to M-series chips. The release also fixes a memory-label regression on 255 GB+ systems.
- **Why it matters:** t3rm1nu55-monitorplus will need equivalent IOReport channel adjustments to correctly report CPU frequencies on MacBook Neo; the macmon fix is the upstream reference implementation to follow.

### Finding 3: arXiv 2604.18788 — ANE used for MoE inference, energy measurements published
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2604.18788
- **Date:** April 20, 2026
- **Summary:** Benazir & Lin present NPUMoE, offloading MoE expert computation to Apple's ANE; benchmarks show 1.32–5.55× prefill-latency and 1.81–7.37× energy reduction across M2/M3/M4 chips. Measurement methodology not fully confirmed but consistent with IOReport Energy Model sampling.
- **Why it matters:** New public energy-per-workload characterization of ANE across three chip generations; confirms IOReport remains the de-facto public measurement surface for ANE power.

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
