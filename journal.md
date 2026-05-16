# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-16 — sweep (4 findings)

> Note: All four findings below predate the last-checked date (2026-04-07) but were missed by the initial seed synthesis. This sweep serves as a catch-up pass over that gap.

### Finding 1: darwin-kperf — Rust crate wrapping kperf/kperfdata for Apple Silicon PMU access
- **Source:** crates.io (HASH / Bilal Mahmoud)
- **URL:** https://crates.io/crates/darwin-kperf
- **Date:** Feb 23, 2026
- **Summary:** `darwin-kperf` is a Rust crate that binds Apple's private `kperf.framework` and `kperfdata.framework` to expose hardware PMU counters (cycles, instructions, cache misses, branch mispredictions) on Apple Silicon with extremely low overhead — the same infrastructure backing Instruments and `xctrace`. A companion crate `darwin-kperf-criterion` provides Criterion.rs integration. Requires root and physical Apple Silicon hardware; no ABI stability guarantee from Apple.
- **Why it matters:** This is a Rust-native kperf FFI that the project's privileged kperf sidecar can directly reference or vendor. Closes the "where's a clean Rust abstraction over kperf?" open question for v2 counter work.

### Finding 2: maderix/ANE — Part 3 (Training) + open-source ANE runtime code
- **Source:** maderix Substack + GitHub
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b · https://github.com/maderix/ANE
- **Date:** March 7, 2026 (Part 3 post); March 3–10, 2026 (repo commits)
- **Summary:** Part 3 of the maderix ANE series demonstrates training a 109M-parameter transformer on the M4 ANE (full forward/backward pass, Adam optimizer) via reverse-engineered `_ANEClient` private APIs. The accompanying `maderix/ANE` GitHub repo now includes INT8 W8A8 quantization achieving 1.88× ANE throughput and a cross-generation ANE benchmark table (March 10 commit). This is the most advanced public ANE access reference implementation to date.
- **Why it matters:** The open-source code is the best available reference for direct ANE private API interaction patterns. No hardware counter access is exposed, but the compile-and-dispatch path is now documented in Rust-adjacent code.

### Finding 3: Orion (arXiv 2603.06728) — systematic ANE characterization including 119-compilation process limit
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 2026
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" finds that deep operation graphs (16–64 ops) achieve 94% ANE utilization, and discovers that the ANE compiler maintains per-process internal state that silently fails all compilations after ~119 attempts per process lifetime. The paper ships an open-source runtime achieving 170+ tok/s for GPT-2 124M on M4 Max hardware. Measurement is purely timing-based — no hardware performance counters are accessed.
- **Why it matters:** The 119-compilation-per-process limit is an undocumented constraint that any long-running monitor process using `_ANEClient` (including a future ANE utilization probe) must work around with multi-process dispatch or explicit reset.

### Finding 4: tzakharko — M5/A19 GPU Neural Accelerator microbenchmark (new hardware surface)
- **Source:** tzakharko.github.io
- **URL:** https://tzakharko.github.io/apple-neural-accelerators-benchmark/ · https://github.com/tzakharko/apple-neural-accelerators-benchmark
- **Date:** ~Nov 2025 (post A19 launch, Sep 19 2025)
- **Summary:** The M5 chip (and Apple A19) introduced dedicated Neural Accelerators integrated into each GPU core — a matrix-multiplication accelerator parallel to and distinct from the traditional ANE block. Tzakharko measured 1024 FLOPS/GPU-core/cycle for FP16 and 2048 OPS/GPU-core/cycle for INT8 via Metal 4 TensorOps timing benchmarks, with optimal 32×32 tile size. The only public access path is Metal 4 TensorOps / Metal Performance Primitives; no raw hardware counter surface has been found.
- **Why it matters:** M5 hardware introduces a second neural accelerator pathway (GPU-embedded) that is neither the traditional ANE nor AMX. The v2 project roadmap needs to distinguish these three accelerators; IOReport may gain new M5 GPU Neural Accelerator energy channels that don't yet have known channel names.

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
