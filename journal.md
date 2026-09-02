# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-09-02 — sweep (5 findings)

### Finding 1: "Apple Neural Engine: Architecture, Programming, and Performance" — comprehensive ANE reverse-engineering paper
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 2026
- **Summary:** A full reverse-engineered account of the ANE built from direct measurement on Apple Silicon and static analysis of AppleNeuralEngine.framework. Documents the datapath, roofline bounds, dispatch route, compiler/program format, weight-compression scheme, and kernel driver internals. This is the most comprehensive public technical specification of the ANE to date.
- **Why it matters:** May expose how to instrument or hook ANE dispatch events — roofline and kernel driver analysis are exactly where counter or utilization signal would surface.

### Finding 2: maderix/ANE — runnable code for ANE training via private APIs
- **Source:** GitHub / maderix
- **URL:** https://github.com/maderix/ANE
- **Date:** ~March 2026 (repo spawned from the Substack series)
- **Summary:** A public GitHub repo implementing transformer training (forward + backward pass) directly on the ANE via `_ANEClient`, `_ANECompiler`, and `_ANEInMemoryModelDescriptor` private APIs, bypassing CoreML entirely. Reports 91 ms/step for Stories110M and 412 ms/step for Qwen3-0.6B. Co-developed with Claude Opus 4.6.
- **Why it matters:** This is the first open-source code artifact that exercises `_ANEClient` directly at training time — it's the closest existing hook point to any future ANE utilization counter.

### Finding 3: "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference"
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 2026
- **Summary:** The first open end-to-end system combining direct ANE execution, a compiler pipeline, and stable multi-step training via Apple's private APIs. Characterizes ANE throughput and power across M-series generations for LLM workloads.
- **Why it matters:** Demonstrates a reproducible pipeline for ANE characterization; the compiler and dispatch analysis may reveal side-channel signals usable for utilization inference.

### Finding 4: "ANEForge: Python for direct computation on the Apple Neural Engine"
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 2026
- **Summary:** A Python library for issuing compute graphs directly to the ANE, developed in parallel with the broader ANE reverse-engineering surge of mid-2026. Provides a higher-level interface over `_ANEClient`-style dispatch.
- **Why it matters:** A Python-level tool for direct ANE dispatch makes it much easier to instrument execution timing and energy deltas experimentally.

### Finding 5: M5 (H17G "Hidra") kpep PMU event database now documented
- **Source:** Web search cross-referencing jiegec/apple-pmu and community reports
- **URL:** https://github.com/jiegec/apple-pmu
- **Date:** 2026 (post-M5 hardware release)
- **Summary:** The M5 chip's kpep event database (`as5.plist`) has been extracted and partially documented. M5 adds `LD_SRC_*` (load data source tracking) and PL2 cache events relative to M4, continuing the pattern of per-generation kpep expansion. No ANE-specific events were identified in the new entries.
- **Why it matters:** The main project's kperf sidecar must handle `as5` CPUID to avoid breaking on M5 hardware; confirms no ANE counter was added in this generation.

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
