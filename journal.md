# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-06 — sweep (6 findings)

### Finding 1: "Apple Neural Engine: Architecture, Programming, and Performance" (arXiv 2606.22283)
- **Source:** arXiv / Spencer Bryngelson (Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.22283 — web edition: https://ane-guide.readthedocs.io
- **Date:** June 2026
- **Summary:** The most comprehensive public reverse-engineering of the ANE to date, based on direct measurement on M1 and M5 plus static analysis of the private runtime, compiler, kernel driver, and firmware. Covers A11–A18 and M1–M5, documenting the datapath and roofline, the dispatch path below CoreML, the e5rt program format, weight compression, and the full IOKit command protocol to the ANE kernel driver.
- **Why it matters:** The kernel driver and command-protocol documentation is the closest anyone has publicly come to the internal ANE monitoring surface; cross-referencing the IOKit call sequence against IOReport Energy Model samples may finally reveal where a utilization counter could be hooked.

### Finding 2: "ANEForge: Python for direct computation on the Apple Neural Engine" (arXiv 2606.17090)
- **Source:** arXiv / Spencer Bryngelson — GitHub: https://github.com/sbryngelson/ANEForge
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 2026
- **Summary:** Companion to 2606.22283 — a Python package that compiles a lazy tensor graph (58 fused ops + 19 native bridge ops) into a single e5rt ANE program and dispatches it through the same ANE daemon stack, without CoreML. Supports forward + backward pass on the ANE; ResNet-18 runs in 0.33 ms. Available on PyPI as `aneforge`.
- **Why it matters:** Provides a clean, scriptable way to generate known-shape ANE workloads and measure IOReport power response — useful for calibrating the power-inference model that t3rm1nu55-monitorplus uses in lieu of direct utilization counters.

### Finding 3: "BaseRT — Apple M5 Neural Accelerators via Metal 4" (arXiv 2607.19438)
- **Source:** arXiv — Waschkowski, Rathnayaka, Wesemann
- **URL:** https://arxiv.org/abs/2607.19438
- **Date:** July 2026
- **Summary:** M5 introduces a new hardware surface: every GPU core now contains a dedicated "Neural Accelerator" exposed through the Metal 4 tensor API. BaseRT adds hand-written Metal 4 GEMM/flash-attention kernels that route LLM inference through these units, achieving 6.4× higher prefill throughput than llama.cpp and 3.9× over MLX on the M5 Pro.
- **Why it matters:** This is a second ANE-class accelerator path on M5, distinct from the classical ANE, accessible via a *documented* Metal 4 API — making it a tractable monitoring target for t3rm1nu55-monitorplus v2 without requiring private-framework reverse engineering.

### Finding 4: darwin-kperf Rust crate — safe kperf.framework FFI bindings
- **Source:** crates.io — https://crates.io/crates/darwin-kperf
- **URL:** https://crates.io/crates/darwin-kperf (docs: https://docs.rs/darwin-kperf)
- **Date:** surfaced in this sweep; publication date on crates.io unconfirmed
- **Summary:** A Rust crate providing safe bindings for Apple's private `kperf.framework` and `kperfdata.framework`, loaded via `dlopen` so there is no link-time private-SDK dependency. Includes `darwin-kperf-sys` for raw FFI and `darwin-kperf-criterion` for Criterion.rs integration. Requires root or `com.apple.private.kernel.kpc` entitlement.
- **Why it matters:** This is a ready-made Rust FFI layer for exactly the kperf access needed in the t3rm1nu55-monitorplus privileged sidecar — substantially reducing the FFI surface we'd have to maintain ourselves.

### Finding 5: SiliconScope — `ri_neural_footprint` public API for per-process ANE usage
- **Source:** GitHub — https://github.com/kennss/SiliconScope
- **URL:** https://github.com/kennss/SiliconScope
- **Date:** surfaced in this sweep; repo creation date unconfirmed
- **Summary:** SiliconScope (sudoless SwiftUI system monitor) exposes per-process ANE activity by calling the **public** `proc_pid_rusage()` with `RUSAGE_INFO_V6` and reading the `ri_neural_footprint` field — no root, no private entitlements. It also uses an IOReport "residency histogram fallback" for bandwidth channels that fail on M4/M5, which broadens chip compatibility.
- **Why it matters:** `ri_neural_footprint` is a completely public API we are not currently using in t3rm1nu55-monitorplus; it gives per-process ANE memory footprint data that could serve as a utilization proxy in the per-process power inference module.

### Finding 6: macmon v0.8.0 — "expose active residency ratios" and per-core metrics
- **Source:** GitHub — https://github.com/vladkens/macmon
- **URL:** https://github.com/vladkens/macmon/releases/tag/v0.8.0
- **Date:** July 24, 2026 (v0.8.0 release); related commits June 8–9, 2026
- **Summary:** macmon v0.8.0 adds "expose active residency ratios" per core, fan speed metrics, and a detailed per-core CPU view. The August 4 fix ("restore per-core metrics on M3 Ultra") confirms the new residency channel works across M3 variants. These ratios are almost certainly sourced from a new IOReport channel read or a different aggregation of existing channels.
- **Why it matters:** macmon is the upstream IOReport reference for t3rm1nu55-monitorplus; we should pull the v0.8.0 residency-ratio approach to ensure our IOReport sampling is up to date with the latest channel names and chip generations.

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
