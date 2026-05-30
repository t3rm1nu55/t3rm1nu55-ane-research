# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-30 — sweep (6 findings)

### Finding 1: Orion — first open end-to-end ANE LLM system with 20-constraint catalog
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" is the most comprehensive public characterization of ANE constraints to date. The paper catalogs 20 restrictions on MIL IR programs (shape limits, memory layout, compilation constraints) discovered by probing _ANEClient/_ANECompiler directly, and demonstrates 170+ tokens/s GPT-2 124M inference on M4 Max. It was missed by the initial seed despite predating it.
- **Why it matters:** The 20-constraint catalog and _ANEClient usage patterns are the deepest public description of the ANE graph execution model; validates IOReport Energy Model as the only viable indirect utilization signal, since no counter path was found.

### Finding 2: maderix — Part 3 of M4 ANE series covers training and M5 hardware data
- **Source:** maderix Substack (tracked source, new installment)
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** Early 2026 (exact date not confirmed)
- **Summary:** Part 3 of the tracked M4 ANE series demonstrates full backpropagation on ANE hardware: forward pass, backward pass, Adam optimizer on 109M-parameter transformer, scaled to Qwen3-0.6B (596M params). The SDPA attention op is decomposed into 3 dispatches because the native ANE SDPA ignores the causal mask. M5 hardware was tested and the M5 ANE is confirmed to be the same H16 family as M4.
- **Why it matters:** M5 ANE = H16 family confirmation reduces uncertainty in multi-generation chip branching; the CPU/ANE dispatch split pattern (weight gradients on CPU, matmuls on ANE) describes how IOReport ANE energy samples map to actual compute load.

### Finding 3: maderix/ANE — open-source repo with working ANE training code and INT8 data
- **Source:** GitHub (new repo, same author as tracked Substack)
- **URL:** https://github.com/maderix/ANE
- **Date:** Last updated March 10, 2026
- **Summary:** MIT-licensed repository implementing transformer training on ANE via _ANEClient/_ANECompiler private APIs, with a C-callable bridge library. INT8 quantization via `constexpr_affine_dequantize` achieves 1.88x throughput improvement. Benchmarks document 18.6 TOPS FP16 on M4 (correcting Apple's marketed 38 TOPS INT8 figure, which assumes 2× FP16 rate).
- **Why it matters:** Working open-source _ANEClient code is directly referenceable; INT8 workloads at 1.88x throughput means ANE IOReport energy-per-token is substantially lower for quantized models — relevant for per-inference power inference.

### Finding 4: CLF3 blog — M3/M4 PMU ESR register encoding changed to 16-bit
- **Source:** blog.clf3.org (new untracked source)
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** 2026 (exact date not confirmed)
- **Summary:** "Utilizing PMU Event Counters on Apple M3 and M4" documents a breaking architecture change: M3/M4 PMESR registers are 64-bit with 16-bit event encoding per slot, versus 8-bit encoding on M1/M2. SYS_APL_PMCR0_EL1 is also repeatedly overwritten by a kernel process — modifications persist only ~100 µs. The post includes working code that correctly handles the M3/M4 register layout.
- **Why it matters:** DIRECTLY actionable for the kperf privileged sidecar: any M1-era PMESR write logic will silently configure wrong events on M3/M4. The sidecar needs a generation-aware PMESR encoding path.

### Finding 5: Mininglamp-AI/cider — M5 INT8 TensorOps via Metal 4 cooperative_tensor unlocked
- **Source:** GitHub (new untracked source)
- **URL:** https://github.com/Mininglamp-AI/cider
- **Date:** 2026 (recent)
- **Summary:** Cider is an MLX-based inference acceleration library that unlocks INT8×INT8→INT32 matrix multiply on M5+ using `mpp::tensor_ops::matmul2d` via Metal 4's `cooperative_tensor` API — hardware not exposed by MLX natively. W8A8 mode achieves 1.4x–1.9x speedup over MLX FP16 on LLM prefill; the INT8 path is conditionally compiled for M5+ only.
- **Why it matters:** The existence of a distinct INT8 hardware execution path on M5 (not present on M4) is a potential new kperf event surface worth probing — if the hardware has a new unit, it likely has new PMU events.

### Finding 6: lambdafoo/mperf — perf-stat-like CLI for Apple Silicon kperf with PET mechanism
- **Source:** lambdafoo.com (new untracked source)
- **URL:** https://lambdafoo.com/posts/2026-03-25-mperf-hardware-counters-macos.html
- **Date:** March 25, 2026
- **Summary:** "Quick Hardware Performance Counters on macOS ARM64" introduces `mperf`, a CLI perf-stat equivalent for Apple Silicon using kperf's Profile Every Thread (PET) mechanism. PET fires a kernel timer that snapshots PMC values for every thread matching a PID filter. The tool enforces a hard 10-event limit (no multiplexing), makes JSON output available for scripting, and works across M1–M4.
- **Why it matters:** PET mechanism is an alternative to per-process attach for the kperf sidecar; the hard 10-event limit and lack of multiplexing is relevant to counter group budget decisions in t3rm1nu55-monitorplus.

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
