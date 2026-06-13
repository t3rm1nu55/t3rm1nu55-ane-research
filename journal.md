# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-13 — sweep (4 findings)

### Finding 1: Orion — first open-source direct ANE runtime (missed by initial seed)
- **Source:** arxiv + GitHub (mechramc/Orion)
- **URL:** https://arxiv.org/abs/2603.06728 · https://github.com/mechramc/Orion
- **Date:** March 6, 2026 (pre-seed; not captured in initial sweep)
- **Summary:** Orion is the first open-source end-to-end system for training and inference directly on the ANE, bypassing CoreML entirely via the private `_ANEClient` and `_ANECompiler` Objective-C APIs. On an M4 Max it achieves 170+ tokens/s for GPT-2 124M inference and trains a 110M-parameter transformer in 22 minutes. The benchmark suite reports TFLOPS, tokens/sec, and an ANE utilization figure (wall-time fraction of `orion_eval` vs. total), which is the closest thing to a real-time ANE utilization metric in open source to date.
- **Why it matters:** Orion's `_ANEClient`/`_ANECompiler` API surface documentation is now the primary open reference for direct ANE programming; any future ANE hardware counter access will likely go through this same call path.

### Finding 2: maderix Part 3 — full transformer training on ANE (missed by initial seed)
- **Source:** maderix Substack
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** March 2026 (pre-seed; not captured in initial sweep)
- **Summary:** Part 3 of the maderix ANE series demonstrates a complete forward pass, backward pass, gradient computation, and Adam optimizer update for a 109M-parameter transformer running entirely on the M4 ANE via reverse-engineered private APIs. The companion GitHub repo (github.com/maderix/ANE) adds SRAM bandwidth probing tools (`sram_bench.m`, `sram_probe.m`) that empirically characterize the ANE's internal memory subsystem.
- **Why it matters:** SRAM bandwidth probing is the closest thing to a hardware-instrumented ANE measurement yet published; the `maderix/ANE` repo should be added to tracked references as a primary source.

### Finding 3: Rigel — Metal 4.1 tensor compute reverse engineering via IOReport energy model
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.12765
- **Date:** June 2026 (new, post-April-7)
- **Summary:** Rigel uses a microbenchmark harness combined with root-free IOReport "Energy Model" sampling to reverse-engineer Apple's Metal 4.1 tensor compute path on an M4 Max GPU, recovering 11 undocumented hardware facts with <2% energy-accounting error. The headline finding is that Metal 4.1 fp8 (E4M3) matmul2d is emulated rather than accelerated in hardware. The authors derive a two-term energy roofline: ~5 pJ/byte for SRAM movement and ~2.7 pJ/FLOP for compute.
- **Why it matters:** The IOReport Energy Model roofline methodology — pairing sustained workloads with cumulative energy deltas to back-calculate operational intensity — is directly transferable to ANE energy inference. This is the most rigorous public demonstration yet that IOReport alone can characterize internal SoC microarchitecture at sub-2% error.

### Finding 4: k06a/macpow — comprehensive IOReport power tree with new channels
- **Source:** GitHub
- **URL:** https://github.com/k06a/macpow
- **Date:** v0.1.19 released May 11, 2026 (new, post-April-7)
- **Summary:** macpow is a Rust/TUI power monitoring tool for Apple Silicon M1–M5 that reads IOReport, SMC, IORegistry, and Mach APIs to show per-component power including GPU SRAM, Media Engine, Camera/ISP, and Fabric — channels that macmon and socpowerbud do not surface. It also handles M5-era IOReport channel renames (MCPU for P-cores, PCPU for Super cores) that break older tools.
- **Why it matters:** macpow's channel list is the most complete public inventory of IOReport Energy Model subscriptions on current hardware; mining it for undocumented ANE sub-channels could reveal finer-grained ANE power breakdowns beyond the single "ANE" accumulator.

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
