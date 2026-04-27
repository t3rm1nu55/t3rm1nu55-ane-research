# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-04-27 — sweep (2 findings)

### Finding 1: Orion — first open catalog of 14 undocumented ANE API constraints (arXiv:2603.06728)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026
- **Summary:** Orion is the first open end-to-end system for direct ANE execution and training that bypasses CoreML entirely via `_ANEClient` and `_ANECompiler`. In the course of building it, the authors cataloged 20 constraints on MIL IR programs, memory layouts, and compilation behavior — 14 of which were previously undocumented. Key discovery: the ANE compiler maintains per-process state and silently fails after approximately 119 compilations, a hard limit not documented anywhere publicly. The paper also shows that deep operation graphs (16–64 ops) are required to approach 94% ANE utilization.
- **Why it matters:** Most complete public documentation of `_ANEClient`/`_ANECompiler` API constraints to date — the constraint catalog and compilation-limit discovery are essential background for any future attempt to wrap ANE dispatch as a utilization proxy in t3rm1nu55-monitorplus v2.

### Finding 2: maderix/ANE — working open-source `_ANEClient` implementation (new reference)
- **Source:** GitHub
- **URL:** https://github.com/maderix/ANE
- **Date:** Created March 2026; 6.6k stars, 18 open PRs as of sweep date
- **Summary:** Companion code to the maderix Substack series (Parts 1–3), this repo provides a working implementation of forward and backward passes on the ANE via reverse-engineered `_ANEClient` and `_ANECompiler` private APIs, with GPU↔ANE zero-copy via IOSurface. It reaches 18.6 TOPS FP16 and 35.1 TOPS INT8 W8A8 on M4, and has trained 109M and 596M parameter transformers from scratch. The repo is not in our tracked reference list and is the most concrete open-source ANE programming artifact to date.
- **Why it matters:** Should be tracked alongside hollance/neural-engine as a live reference for the private ANE API surface; the IOSurface zero-copy pattern is a potential low-overhead hook point for utilization inference.

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
