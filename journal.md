# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-04-21 — sweep (3 findings)

> Note: This is the first live sweep. The April 7 entry was the initial seed, not an automated run.
> All three findings predate April 7 but were absent from the seed; they are logged here to close those gaps.

### Finding 1: maderix ANE Part 3 "Training" + open-source `maderix/ANE` repo

- **Source:** maderix Substack / GitHub
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b · https://github.com/maderix/ANE
- **Date:** 2026-03-07 (Part 3 post); repo active through 2026-03-10
- **Summary:** The third installment of the maderix M4 ANE series demonstrates a full forward + backward pass (109M-parameter transformer, Adam optimizer) executed on the ANE via reverse-engineered private APIs — hardware Apple ships for inference only. Accompanying open-source code (`maderix/ANE`) provides a C-callable bridge that resolves `_ANEClient`, `_ANECompiler`, and the newly documented `_ANEPerformanceStats` private class at runtime. `_ANEPerformanceStats` is an undocumented class in `AppleNeuralEngine.framework` and is the first publicly identified symbol that may surface ANE-internal performance data to the host.
- **Why it matters:** `_ANEPerformanceStats` is the only currently known private symbol plausibly exposing ANE hardware metrics; it warrants reverse-engineering to determine if it wraps a counter or is purely software-side bookkeeping.

### Finding 2: Orion — Characterizing and Programming Apple's Neural Engine (arXiv 2603.06728)

- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** 2026-03-06
- **Summary:** Academic paper describing a full programming model for the ANE targeting LLM training and inference. Key empirical results: deep operation graphs achieve 94% ANE utilization (measured via benchmark proxying against IOReport power, not hardware counters); the ANE compiler silently fails after ~119 compilations per process, requiring an `exec()` restart strategy (~50 ms cost per restart). Achieves GPT-2 124M inference at 170+ tokens/sec on M4.
- **Why it matters:** The ~119-compilation limit is a hard constraint any ANE utilization sampler must account for; the 94% utilization figure confirms IOReport power-delta remains the only viable proxy for ANE activity from outside the private API surface.

### Finding 3: XTC Research Platform — cross-platform KPerf harness for Apple Silicon (arXiv 2512.16512)

- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2512.16512
- **Date:** 2025-12
- **Summary:** XTC is a research platform for AI workload optimization that, as part of its measurement harness, uses Apple's undocumented `kperf`/`kperfdata` framework and the `/usr/share/kpep` database on macOS to access hardware performance counters on Apple Silicon CPUs — the first published cross-platform counter harness that explicitly targets this path. The paper validates counter reads against x86, non-Apple ARM, and NVIDIA GPU targets in the same framework.
- **Why it matters:** XTC's macOS counter path is a working, citable open reference for the kperf FFI approach used in the t3rm1nu55-monitorplus privileged sidecar; the kpep-database event-translation pattern is directly reusable.

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
