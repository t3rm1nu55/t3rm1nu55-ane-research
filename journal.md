# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-10 — sweep (3 findings)

### Finding 1: Orion — open-source ANE training+inference runtime
- **Source:** arXiv + github.com/mechramc/Orion
- **URL:** https://arxiv.org/abs/2603.06728 / https://github.com/mechramc/Orion
- **Date:** March 2026
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" (arXiv:2603.06728) is the first open end-to-end system for LLM training and inference directly on the ANE, bypassing CoreML entirely via `_ANEClient` and `_ANECompiler`. The companion GitHub repo contains `docs/ane_constraints.md` cataloging 14 documented ANE operational constraints (including the ~119 compile-per-process limit and minimum 49 KB IOSurface allocation floor). ANE "utilization" in Orion is measured as wall-clock time fraction inside `orion_eval`, not via hardware counters — confirming no counter path has been found yet.
- **Why it matters:** Current upper bound on public ANE characterization; the constraint catalog and expanded private API surface are required reading before any counter-exposure attempt.

### Finding 2: maderix/ANE — new public repo, new private class names
- **Source:** github.com/maderix/ANE
- **URL:** https://github.com/maderix/ANE
- **Date:** February–March 2026
- **Summary:** maderix published a companion GitHub repo alongside Part 3 of the ANE Substack series (training a 109M-parameter transformer on ANE with full forward+backward pass). The `api_exploration.m` file surfaces three undocumented private classes not previously in the public symbol table: `_ANEInMemoryModelDescriptor`, `MLANEEngine`, and `MLMILComputeEngine`. No counter or utilization data is exposed; the file focuses on model-loading path introspection.
- **Why it matters:** `MLANEEngine` is a candidate symbol for future runtime introspection for utilization or counter hooks; adds to the known-private-API corpus.

### Finding 3: macmon v0.7.x — M5 breaks IOReport voltage-state key names
- **Source:** github.com/vladkens/macmon releases
- **URL:** https://github.com/vladkens/macmon/releases/tag/v0.7.0
- **Date:** April 1–May 2, 2026
- **Summary:** macmon v0.7.0 (April 1) through v0.7.2 (May 2) added M5 compatibility by fixing a crash caused by Apple renumbering IOReport voltage-state keys on M5 Max, and adapting to a new E/P/S core label scheme. The "S" core label is new — M5 introduces a third architectural core class absent from M1–M4.
- **Why it matters:** Directly actionable for t3rm1nu55-monitorplus: the same crash will occur on M5 hardware. Voltage-state key parsing needs conditional logic or dynamic key discovery keyed on chip generation; the new S-core class also needs a display/metrics path.

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
