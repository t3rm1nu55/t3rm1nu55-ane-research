# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-22 — sweep (2 findings)

### Finding 1: Orion — first formal arXiv paper characterizing ANE with 94% utilization metric

- **Source:** arXiv + mechramc/Orion (GitHub)
- **URL:** https://arxiv.org/abs/2603.06728 · https://github.com/mechramc/Orion
- **Date:** March 6, 2026 (published before this sweep window; not captured in initial seed)
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" is the first formal academic treatment of direct ANE execution without CoreML. It reports 94% ANE utilization for deep operation graphs (16–64 ops), extends the public ANE constraint catalog to 20 entries (14 newly documented), and discovers a per-process compiler state limit of ~119 compilations before silent failure.
- **Why it matters:** The 94% utilization methodology is the closest thing to a real ANE utilization metric in the public record; the Orion codebase (MIT license) is the first open-source reference for direct ANE scheduling without CoreML overhead.

### Finding 2: maderix Part 3 (Training) + maderix/ANE open-source codebase

- **Source:** maderix Substack (Part 3) + maderix/ANE (GitHub)
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b · https://github.com/maderix/ANE
- **Date:** March 7, 2026 (follow-up to tracked Part 2; not captured in initial seed)
- **Summary:** Part 3 completes the maderix M4 ANE series with the first public full backward pass on ANE hardware, including gradient computation and Adam optimizer updates for a 109M-parameter transformer trained from scratch on inference-only hardware. The accompanying GitHub repo (MIT license) is the first open-source implementation of direct `_ANEClient` access for both forward and backward ANE execution.
- **Why it matters:** The maderix/ANE codebase is the most immediately actionable reference for hooking ANE execution activity in t3rm1nu55-monitorplus v2; it demonstrates that `_ANEClient`-level dispatch detection is feasible without CoreML.

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
