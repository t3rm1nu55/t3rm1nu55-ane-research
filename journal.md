# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-28 — sweep (3 findings)

### Finding 1: M3/M4 PMU ESR register uses 16-bit event encoding (vs 8-bit on M1/M2)
- **Source:** ClF3's blog
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** 2025–2026 (exact date not publicly indexed; surfaced May 2026)
- **Summary:** "Utilizing PMU Event Counters on Apple M3 and M4" documents a register-format change: on M3/M4, the PMU event select register (ESR) is 64-bit with 16 bits allocated per event, versus 8 bits per event on M1/M2. This changes how event codes must be packed when programming kperf counters on newer chips. No new AMX- or ANE-specific events are reported, but the structural change affects all event-slot programming.
- **Why it matters:** The kperf privileged sidecar will need chip-generation-aware ESR packing logic to correctly program M3/M4 counters; using M1/M2 layout on M3/M4 will silently misconfigure events.

### Finding 2: Orion — first open end-to-end ANE runtime with compiler, bypasses CoreML
- **Source:** arXiv 2603.06728 + github.com/mechramc/Orion
- **URL:** https://arxiv.org/abs/2603.06728 / https://github.com/mechramc/Orion
- **Date:** March 2026
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" is the first complete open system to bypass CoreML and invoke the ANE directly via private `_ANEClient`/`_ANECompiler` APIs. It lowers a graph IR through five optimization passes to ANE-native MIL format, achieves 170+ tok/s on GPT-2 124M (M4 Max), and extends the maderix work to stable LLM training with checkpoint resume. The MIT-licensed companion repo is a working implementation.
- **Why it matters:** Orion's "ANE utilization" metric is almost certainly wall-clock–derived (TFLOPS from timing ÷ peak TFLOPS), not a hardware counter — confirming the counter gap persists — but its `_ANEClient` compiler pipeline is now the most complete public reference for the ANE private API surface.

### Finding 3: maderix/ANE repo — 40+ private ANE class catalog + Part 3 training post
- **Source:** github.com/maderix/ANE + maderix Substack Part 3
- **URL:** https://github.com/maderix/ANE / https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** March 2–10, 2026 (repo); March 7, 2026 (Part 3 post)
- **Summary:** maderix published a companion GitHub repo cataloging 40+ private classes in `AppleNeuralEngine.framework` — including `_ANEClient`, `_ANEModel`, `_ANERequest`, `_ANEIOSurfaceObject` — alongside an `api_exploration.m` that demonstrates direct ANE invocation without CoreML. Part 3 demonstrates full transformer training (forward + backward, Adam, 109M params) on hardware designed for inference; reported ANE throughput is only ~5–9% of peak TFLOPS, suggesting large scheduling overhead.
- **Why it matters:** The 40+ class list in `api_exploration.m` is now the most complete public symbol table for `AppleNeuralEngine.framework` and is the authoritative reference if a future t3rm1nu55-monitorplus ANE probe is attempted via private APIs.

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
