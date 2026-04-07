# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-04-07 — sweep (4 findings)

### Finding 1: ClF3 — PMU event register format changes on M3 and M4
- **Source:** ClF3's blog
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** 2025/2026 (exact publication date not retrieved)
- **Summary:** Documents a breaking change in Apple Silicon PMU register layout: on M3 and M4, the ESR configuration registers are 64-bit with 16 bits allocated per event, compared to M1/M2's 8 bits per event. Also notes that `SYS_APL_PMCR0_EL1` gets overwritten by the kernel within ~100 µs, making sustained PMU sampling impossible on stock macOS without a patched kernel. Provides the exact SYS_APL_PMCR0/PMCR1 bit patterns for enabling PMC2–9 at EL0/EL1.
- **Why it matters:** The kperf sidecar in t3rm1nu55-monitorplus must handle M3/M4 register format differently from M1/M2; the 100 µs window also constrains the sampling strategy for the privileged helper.

### Finding 2: Orion — first open end-to-end ANE LLM runtime
- **Source:** arXiv + GitHub
- **URL:** https://arxiv.org/abs/2603.06728 / https://github.com/mechramc/Orion
- **Date:** March 2026
- **Summary:** Academic paper + working codebase that bypasses CoreML entirely via Apple's private `_ANEClient` and `_ANECompiler` APIs. Implements a five-pass compiler from a graph IR down to ANE-native MIL, with IOSurface-backed zero-copy tensor I/O. Key innovation is "delta compilation": after each training step, compiled ANE programs are patched in-place rather than fully recompiled, cutting recompilation from 4,200 ms to 494 ms (8.5×), yielding a 3.8× end-to-end training speedup on M4 Max (170+ tokens/s inference on GPT-2 124M). The "94% ANE utilization" figure is derived from throughput benchmarking, not hardware counters.
- **Why it matters:** Establishes the complete `_ANEClient`/`_ANECompiler` private API surface in a single auditable codebase; any future ANE utilization metric hook will need to instrument this layer or its kernel counterpart.

### Finding 3: maderix Part 3 + maderix/ANE repo — training + INT8 on ANE
- **Source:** maderix Substack + GitHub
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b / https://github.com/maderix/ANE
- **Date:** March 2026 (repo commits: 2026-03-06 to 2026-03-10)
- **Summary:** Part 3 of the maderix M4 ANE series covers full transformer training on the ANE (forward pass, backward pass, gradient computation) on a 109M-parameter model designed for inference. The companion GitHub repo has since added INT8 W8A8 quantization via `quantize`/`dequantize` MIL ops, achieving 1.88× ANE throughput improvement, and multi-model support including Qwen3-0.6B with GQA.
- **Why it matters:** Shows the maderix work is maturing into production tooling; INT8 results confirm that compile-time weight baking (not runtime counters) drives ANE optimization today.

### Finding 4: lauka — Apple Silicon PMU counter CLI
- **Source:** GitHub
- **URL:** https://github.com/verte-zerg/lauka
- **Date:** Created 2026-01-07
- **Summary:** A minimal CLI built on the reverse-engineered kperf API (ibireme's gist) that records named Apple Silicon PMU counters (cycles, instructions, branch mispredictions, L1D misses, etc.) across multiple commands, computes per-metric statistics (mean/stddev/min/max/outliers), and emits a delta column comparing each command against a baseline. Requires `sudo`. Supports at least M3.
- **Why it matters:** New reference implementation for kperf tooling that is more ergonomic than the ibireme gist; the `lauka counters --details` command is a useful enumeration of available event names per chip generation.

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
