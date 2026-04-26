# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-04-26 — sweep (2 findings)

> Note: both findings were published in March 2026 (before the April 7 last-checked date) but were absent from the initial synthesis. Logged here on first detection.

### Finding 1: Orion — first open end-to-end ANE system and constraint catalog
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026
- **Summary:** First open paper+system that bypasses CoreML entirely via `_ANEClient` and `_ANECompiler` private APIs, achieving 170+ tokens/s for GPT-2 124M on M4 Max and stable 1,000-step training of a 110M-parameter transformer. Catalogs 20 ANE compiler constraints (14 newly documented), including a per-process ~119-compilation hard limit before silent failure. Reduces recompilation overhead 8.5x via patch-and-reload rather than full recompile.
- **Why it matters:** Most systematic public mapping of `_ANEClient`/`_ANECompiler` interfaces to date; the constraint catalog is the best reference for safe utilization-probing or counter-sidecar code in t3rm1nu55-monitorplus.

### Finding 2: maderix ANE Part 3 (Training) and INT8 W8A8 repo commit
- **Source:** maderix Substack / GitHub (maderix/ANE)
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b (Part 3); https://github.com/maderix/ANE (repo)
- **Date:** March 7, 2026 (Part 3 post); March 10, 2026 (INT8 commit `"Add INT8 W8A8 support: 1.88x ANE throughput via quantize/dequantize MIL ops"`)
- **Summary:** Part 3 demonstrates a full transformer forward+backward pass on the M4 ANE via `_ANEClient`, reporting ~11.2% ANE utilization during training — measured via power-delta, not hardware counters. The companion repo added INT8 W8A8 quantization via MIL `quantize`/`dequantize` ops, yielding 1.88x throughput uplift.
- **Why it matters:** Confirms no counter-level utilization surface has appeared; power-indirection via IOReport Energy Model remains the state of the art for ANE metrics in t3rm1nu55-monitorplus v1.

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
