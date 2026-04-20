# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-04-20 — sweep (4 findings)

Four pieces of work published in February–March 2026 were missed in the initial April 7 seed. All confirmed relevant to ANE/AMX characterization or kperf PMU access on Apple Silicon.

### Finding 1: Orion — first open end-to-end ANE training and inference system
- **Source:** arXiv (2603.06728); HN discussion surfaced April 18 2026
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026 (submitted); HN discussion April 18, 2026
- **Summary:** Orion is the first fully open system for LLM training and inference directly on the ANE, bypassing CoreML entirely via `_ANEClient` + `_ANECompiler` private APIs. It introduces a five-pass compiler pipeline, IOSurface-backed zero-copy tensor I/O, and delta compilation (8.5× speedup over per-step recompilation). On M4 Max it achieves 170+ tokens/s for GPT-2 124M inference and trains a 110M-parameter transformer in 22 minutes. The paper also documents a hard per-process ANE state limit of ~119 compilations before silent failures begin.
- **Why it matters:** Provides the most detailed public mapping of the `_ANEClient`/`_ANECompiler` API surface to date; the ~119-compile state limit is a novel side-channel that could proxy ANE activity without hardware counters.

### Finding 2: maderix — Part 3, full transformer training on ANE
- **Source:** maderix Substack
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** March 7, 2026
- **Summary:** Third entry in the tracked maderix M4 ANE series; demonstrates the first public full backward pass + Adam optimizer update running on ANE hardware via reverse-engineered `_ANEClient`. Scales to Qwen3-0.6B (596M parameters). Companion GitHub repo `maderix/ANE` ships working code. Key insight: expressing computation as 1×1 convolutions rather than matrix multiply unlocks dramatically higher ANE throughput.
- **Why it matters:** Extends the tracked `_ANEClient` private API surface with gradient/optimizer operations; `maderix/ANE` GitHub repo should be added to tracked references.

### Finding 3: darwin-kperf-criterion — Rust crate for Apple Silicon PMU counter access
- **Source:** crates.io / docs.rs
- **URL:** https://docs.rs/crate/darwin-kperf-criterion/latest
- **Date:** February 23, 2026 (v0.1.0)
- **Summary:** A Rust crate that integrates Apple Silicon hardware PMU counters (via the private kperf/kpc interface) with the Criterion.rs benchmarking framework. Supports M1–M5. Requires root or the `com.apple.private.kernel.kpc` entitlement; falls back to wall-clock on non-macOS. Exposes 2 fixed counters (cycles, instructions) plus 8 configurable hardware counter slots as a `Measurement` type.
- **Why it matters:** This is a Rust-native kperf FFI already written and published — directly applicable to t3rm1nu55-monitorplus's planned kperf sidecar; worth reviewing as a vendoring or reference candidate before writing new FFI code.

### Finding 4: M5 ANE/AMX roofline analysis — first M5 quantitative baselines
- **Source:** michaelstinkerings.org
- **URL:** https://www.michaelstinkerings.org/apple-m5-gpu-roofline-analysis/
- **Date:** March 16, 2026
- **Summary:** Roofline performance sweep of M5 across CPU-AMX, GPU, and ANE compute units. ANE peaks at 7,732 GFLOPS FP16 at batch=256, exceeding both GPU (6,004 GFLOPS) and CPU-AMX (225 GFLOPS at batch=1 decode). Measurement is via Core ML interface, not hardware counters. AMX dominates at small batch sizes (decode), while ANE dominates at large batch sizes (prefill).
- **Why it matters:** Establishes M5 ANE/AMX throughput baselines useful for calibrating power-inferred utilization estimates; the AMX vs ANE crossover point at different batch sizes is new data for t3rm1nu55-monitorplus's per-process power inference.

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
