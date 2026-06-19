# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-19 — sweep (4 findings)

### Finding 1: Orion — First Open End-to-End ANE Compiler and Training Runtime
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 2026
- **Summary:** Orion (Kumaresan) is the first open system combining direct ANE execution, a five-pass compiler from graph IR to ANE-native MIL, and multi-step training with checkpoint resume — all bypassing CoreML entirely via `_ANEClient` and `_ANECompiler` private APIs. It catalogs 20 ANE execution constraints and achieves 170+ tokens/s GPT-2 inference and a 3.8× training speedup on M4 Max. The companion maderix/ANE code repository (Finding 3) implements the same approach independently.
- **Why it matters:** Most thorough public documentation of ANE compiler internals to date; the 20-constraint catalog and IOSurface-backed tensor I/O details are the most complete public map of the ANE dispatch surface, informing any future ANE telemetry hooking.

### Finding 2: darwin-kperf — Rust kperf/kpc Crate Published to crates.io
- **Source:** crates.io
- **URL:** https://crates.io/crates/darwin-kperf
- **Date:** February 23, 2026 (v0.1.0, author: Bilal Mahmoud / HASH)
- **Summary:** A Rust crate wrapping Apple's private `kperf.framework` and `kperfdata.framework` with a safe Rust API for CPU PMC counter configuration and read-back; companion `darwin-kperf-sys` provides raw FFI bindings and `darwin-kperf-criterion` integrates with Criterion.rs. Requires root or the `com.apple.private.kernel.kpc` entitlement, consistent with how the main project's privileged sidecar already operates.
- **Why it matters:** Directly actionable — t3rm1nu55-monitorplus's kperf sidecar FFI should be diffed against this crate's binding surface; adopting or forking it would reduce maintenance burden and give us a community-validated ABI.

### Finding 3: maderix/ANE — Code Repository for Reverse-Engineered ANE Training
- **Source:** GitHub
- **URL:** https://github.com/maderix/ANE
- **Date:** Early 2026 (companion to March 2026 Substack series)
- **Summary:** Manjeet Singh published a runnable code repository implementing transformer forward and backward passes directly on the ANE via reverse-engineered `_ANEClient`, `_ANECompiler`, and `_ANEInMemoryModelDescriptor` private APIs, successfully training a 109M-parameter Llama-2-architecture model. The repo is MIT-licensed with no Apple proprietary code and is distinct from the Substack writeup already tracked in references.md.
- **Why it matters:** Provides the most concrete inspectable reference for direct ANE dispatch at the API level, including how `IOSurface`-backed zero-copy buffers are managed — directly relevant to any future ANE utilization sampling probe.

### Finding 4: NPUMoE — Mixture-of-Experts LLM Inference Characterizes ANE Dispatch Overhead (arXiv 2604.18788)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2604.18788
- **Date:** April 20, 2026
- **Summary:** NPUMoE offloads dense, static expert computations to the ANE while routing dynamic operations to CPU/GPU, achieving 1.32×–5.55× latency reduction and 1.81×–7.37× energy efficiency gain on M2 Ultra. The paper explicitly characterizes the ANE's minimum viable kernel size (~1 ms dispatch floor) and the penalty for dynamic tensor shapes, and uses the IOReport Energy Model channel as ground-truth power measurement.
- **Why it matters:** Confirms and quantifies the ANE dispatch overhead that would degrade any sub-millisecond utilization sampling scheme; the IOReport energy-channel methodology matches what t3rm1nu55-monitorplus already uses for power inference.

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
