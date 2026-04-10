# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-04-10 — sweep (3 findings)

### Finding 1: maderix Part 3 — Full transformer training on the M4 ANE
- **Source:** maderix Substack (tracked)
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** 2026-03-07
- **Summary:** Third installment of the M4 ANE reverse-engineering series. Demonstrates a complete training loop — forward pass, backward pass, gradient computation, AdamW optimizer updates — running natively on the ANE via `_ANEClient`/`_ANECompiler` private APIs, bypassing CoreML entirely. A companion open-source repo (`maderix/ANE`) was published simultaneously and shows active development through 2026-03-10, including INT8 W8A8 quantisation (1.88× ANE throughput via MIL `quantize`/`dequantize` ops) and multi-model benchmarks. The initial synthesis cited Part 2 as "the current best public characterisation" but missed Part 3, which significantly advances the known API surface.
- **Why it matters:** Confirms `_ANEClient` is stable enough for training-class dispatch; the maderix/ANE repo is now a live reference implementation of the private API call sequence that could inform an "ANE busy" heuristic via process-dispatch monitoring.

### Finding 2: Orion (arxiv:2603.06728) — First open end-to-end ANE LLM system
- **Source:** arXiv (tracked search string: "Apple Neural Engine" AND "counter" OR "utilization" OR "benchmark")
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** 2026-03-06
- **Summary:** Academic paper describing Orion, the first open system for end-to-end ANE LLM training and inference that bypasses CoreML via `_ANEClient`/`_ANECompiler`. It documents 20 ANE programming constraints (14 newly discovered MIL IR and memory constraints), a compiler state limit of ~119 `ANECCompile()` calls per process before silent failure (addressed in Orion v2 via delta compilation), and GPT-2 124M inference at 170+ tok/s. The "94% ANE utilisation" figure is derived from throughput benchmarking against peak, not from a hardware counter.
- **Why it matters:** Most thorough public documentation of the `_ANEClient`/`_ANECompiler` API surface to date; the compilation counter limit is a new operational constraint relevant to any tool that probes ANE activity by driving test compilations.

### Finding 3: ClF3 blog — M3/M4 PMU ESR register encoding differs from M1/M2
- **Source:** https://blog.clf3.org/post/pmu-event-counters/ (new, not yet tracked)
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** Unknown (appeared alongside bugsiki.dev in tracked LKML search results)
- **Summary:** Documents that the PMU ESR registers on M3 and M4 are 64-bit (vs 48-bit on M1/M2), and that each kperf event selector occupies 16 bits (vs 8 bits on M1/M2). Counter configuration code that hardcodes M1/M2 shift/mask arithmetic will silently misconfigure event selectors on M3/M4, likely producing zeroed or garbage counter readings without any error.
- **Why it matters:** Directly actionable for the kperf privileged sidecar: the event-encoding layer must branch on chip generation or it will produce incorrect counter reads on M3/M4 hardware. Opening a tracking issue on the main project.

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
