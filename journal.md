# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-09-03 — sweep (4 findings)

### Finding 1: ANEForge — Python library for direct ANE computation without CoreML
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 12, 2026
- **Summary:** Spencer Bryngelson (Georgia Tech) released ANEForge, a Python package that compiles a lazy tensor graph of 58 fused operators directly to a single ANE execution program, bypassing CoreML entirely via the same private `_ANEClient`/`_ANECompiler` APIs that maderix reverse-engineered. Training (forward + backward + Adam) runs on ANE; a ResNet-18 forward pass takes 0.33 ms. Installable from PyPI; companion GitHub at `sbryngelson/ANEForge`.
- **Why it matters:** First publicly installable tool for direct ANE graph programming; establishes a stable open interface we can study for any utilization or counter exposure hooks.

### Finding 2: "Apple Neural Engine: Architecture, Programming, and Performance" — comprehensive public reference (June 2026)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 21, 2026
- **Summary:** Companion paper by Bryngelson covering ANE architecture across every chip generation from A11 through A18 and M1 through M5, with per-chip operation support tables and a programming model reference. Most complete public documentation of ANE internals published to date; companion guide at `ane-guide.readthedocs.io`.
- **Why it matters:** Authoritative chip-by-chip operation and capability matrix; directly relevant for deciding which chip-generation gating logic to implement when ANE utilization inference is added.

### Finding 3: M4+ Apple chips use ARM SME, not Apple AMX — microarchitecture boundary confirmed
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 24, 2026
- **Summary:** Deyvik Bhan (Georgia Tech) microbenchmarks M1 AMX for LLM GEMM, characterizing it as load-issue bound at ~610–680 GFLOPS under interleaved loads. The paper explicitly confirms that M4 and later SoCs replaced Apple's proprietary AMX with the publicly-documented ARM Scalable Matrix Extension (SME); M1–M3 use Apple AMX, M4+ use ARM SME.
- **Why it matters:** Kills the assumption that "AMX counter" work generalizes across all M-series chips. AMX event discovery efforts apply only to M1–M3; on M4+ the SME has ARM-standard PMU events, making counter access there much more tractable.

### Finding 4: darwin-kperf — Rust FFI crate wrapping Apple's private kperf framework
- **Source:** crates.io / docs.rs
- **URL:** https://crates.io/crates/darwin-kperf
- **Date:** Published 2026 (exact date unconfirmed; absent from April 2026 sweep)
- **Summary:** `darwin-kperf` by Bilal Mahmoud / HASH wraps Apple's private `kperf.framework` and `kperfdata.framework` for Rust via `dlopen` at runtime, exposing hardware PMU counters (cycles, instructions, cache misses, branch mispredictions) with no link-time dependency on private headers. Companion `darwin-kperf-criterion` integrates with Criterion benchmarks. Requires root and physical Apple Silicon hardware; no ABI stability guarantee.
- **Why it matters:** Directly eliminates the need to write our own unsafe kperf FFI in t3rm1nu55-monitorplus; the privileged sidecar could vendor this crate instead of hand-rolling the dlopen binding.

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
