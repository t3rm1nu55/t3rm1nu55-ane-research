# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-20 — sweep (3 findings)

### Finding 1: Orion — first systematic ANE characterization and direct programming paper
- **Source:** arXiv (missed by seed synthesis; not yet in references)
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026
- **Summary:** Ramchand Kumaresan's "Orion" is the first published end-to-end system for direct ANE programming that bypasses CoreML entirely via the private `_ANEClient` and `_ANECompiler` APIs. It catalogs 20 previously undocumented MIL IR restrictions, documents the 119-compilation-per-process hard limit, and demonstrates a weight-patching technique that bypasses `ANECCompile()` for weight updates, cutting recompilation from 4,200 ms to 494 ms (8.5×). On M4 Max, achieves 170+ tokens/s GPT-2 124M inference and claims 94% ANE utilization on deep operation graphs (16–64 ops); measurement methodology for that utilization figure is unclear from the abstract.
- **Why it matters:** The 20-restriction catalog and compilation-limit discovery are the most systematic public characterization of the ANE execution model to date; the claimed "94% utilization" figure warrants scrutiny — if it is power-derived it is already reproducible via IOReport, if it is a hardware counter it is a breakthrough.

### Finding 2: maderix ANE GitHub repo and Part 3 (training on ANE)
- **Source:** github.com/maderix/ANE + maderix Substack Part 3 (missed by seed synthesis)
- **URL:** https://github.com/maderix/ANE  |  https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** March 3, 2026
- **Summary:** The seed captured maderix's Part 2 (ANE benchmarks) but missed Part 3 and the companion GitHub repo. Part 3 documents the first publicly verified backpropagation pass on the ANE, working around the 119-compile limit via weight-blob hot-patching, and a full Adam optimizer loop for a 109M-parameter transformer on TinyStories. The `maderix/ANE` repo contains reference code for the `_ANEClient`/`_ANECompiler` FFI call sequence.
- **Why it matters:** The hot-patching technique and `_ANECompiler` call sequence in the repo are the most complete public reference for the ANE private API surface; useful context if t3rm1nu55-monitorplus ever attempts to hook ANE dispatch for utilization inference.

### Finding 3: darwin-kperf — native Rust crate for kperf/kpc counter access
- **Source:** crates.io (missed by seed synthesis; not yet in references)
- **URL:** https://crates.io/crates/darwin-kperf  |  https://lib.rs/crates/darwin-kperf-criterion
- **Date:** February 23, 2026
- **Summary:** `darwin-kperf` (v0.1.0, author: Bilal Mahmoud / HASH) provides a native Rust FFI wrapper over Apple's private `kperf.framework` and `kperfdata.framework`, exposing programmable hardware performance counters (cycles, instructions, cache misses, branches) with low overhead. Companion crate `darwin-kperf-criterion` integrates this as a Criterion.rs `Measurement` backend. Requires root or `com.apple.private.kernel.kpc` entitlement; ABI stability not guaranteed across macOS versions.
- **Why it matters:** Directly usable by t3rm1nu55-monitorplus as a maintained Rust-native kperf layer; the privileged sidecar already has the required entitlement, so this could replace any hand-rolled kperf FFI in the project.

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
