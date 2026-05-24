# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-24 — sweep (5 findings)

*Note: findings 1–4 predate the last-check date of 2026-04-07 but were absent from the initial seed. Finding 5 is the only genuinely post-April-7 entry.*

### Finding 1: Orion — first systematic ANE characterization for LLM workloads
- **Source:** arXiv (Ramchand Kumaresan)
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026 (seed catch-up)
- **Summary:** The Orion paper achieves 94% ANE utilization using deep operation graphs (16–64 ops) and catalogs 20 ANE constraints — 14 newly discovered — including the per-process 119-compilation cap and memory/I/O layout rules. It also introduces a compiler that generates ANE programs directly without CoreML. First academic paper to characterize ANE comprehensively for LLM training and inference.
- **Why it matters:** The 94% utilization ceiling and constraint catalog give us the best public model of when the ANE is saturated vs. idle — critical context for interpreting IOReport energy deltas as a utilization proxy.

### Finding 2: maderix Part 3 — transformer training on ANE, open-source code
- **Source:** maderix Substack / GitHub
- **URLs:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b · https://github.com/maderix/ANE
- **Date:** March 7, 2026 (seed catch-up — seed tracked Parts 1 & 2 only)
- **Summary:** Part 3 demonstrates full transformer training (forward + backward pass, Adam optimizer, 109M parameters) running natively on the M4 ANE. The accompanying open-source repo benchmarks ANE dispatch latency at ~119 µs + bytes/78 GB/s and documents a current utilization ceiling of just 5–9% of theoretical peak. INT8 W8A8 achieves 35.1 TOPS.
- **Why it matters:** The 5–9% utilization figure confirms that power-based inference (IOReport energy deltas) is the practical proxy — raw counter-derived utilization would show similarly low numbers given current dispatch overhead.

### Finding 3: darwin-kperf — Rust crate providing kperf/kperfdata bindings
- **Source:** crates.io
- **URL:** https://crates.io/crates/darwin-kperf
- **Date:** February 23, 2026 (seed catch-up)
- **Summary:** `darwin-kperf` is a published Rust crate binding Apple's private kperf and kperfdata frameworks for reading hardware PMU counters on Apple Silicon. Companion crate `darwin-kperf-criterion` integrates with Criterion benchmarks. Requires root; stable across M1–M4.
- **Why it matters:** The main project is Rust and currently rolls its own kperf FFI in the privileged sidecar; `darwin-kperf` could replace that with an off-the-shelf crate — should evaluate for adoption or at least reference.

### Finding 4: Apple replacing Core ML with Core AI at WWDC 2026
- **Source:** 9to5Mac (multiple corroborating outlets)
- **URL:** https://9to5mac.com/2026/03/01/apple-replacing-core-ml-with-modernized-core-ai-framework-for-ios-27-at-wwdc/
- **Date:** March 1, 2026 (seed catch-up; WWDC announcement is June 2026)
- **Summary:** Credible reports confirm Apple will replace Core ML with a new "Core AI" framework in iOS 27 / macOS 26 at WWDC 2026, opening the framework to third-party models and MCP integration. No details yet on whether any new ANE performance monitoring or utilization APIs are included.
- **Why it matters:** A new public framework could expose ANE utilization metrics or deprecate the private `_ANEClient`/`_ANECompiler` APIs that current RE work relies on. WWDC 2026 is June 2026 — the announced API surface must be reviewed immediately on release.

### Finding 5: NPUMoE — ANE offload for MoE LLMs with measured energy efficiency
- **Source:** arXiv (Afsara Benazir, Felix Xiaozhu Lin)
- **URL:** https://arxiv.org/abs/2604.18788
- **Date:** April 20, 2026 *(first finding genuinely post-last-check)*
- **Summary:** NPUMoE is a runtime inference engine that offloads dense, static expert computation in Mixture-of-Experts LLMs to Apple Silicon ANE while CPU/GPU handles dynamic routing ops. Achieves 1.32–5.55× latency reduction and 1.81–7.37× energy efficiency improvement across M-series devices, measured via CPU performance counters and IOReport-style energy metrics.
- **Why it matters:** Validates IOReport energy measurement as the authoritative method for ANE activity tracking; the paper's counter + energy measurement methodology is directly reusable. Also signals growing academic use of direct ANE access, increasing the probability of counter-exposure research emerging.

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
