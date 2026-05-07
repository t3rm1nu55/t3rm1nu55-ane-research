# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-07 — sweep (2 findings)

Both findings predate this sweep's window but were absent from the initial seed; surfaced by re-running the arXiv and blog search strings against the full tracked-source list.

### Finding 1: maderix/ANE — open-source `_ANEClient`/`_ANECompiler` training repo

- **Source:** GitHub — maderix/ANE
- **URL:** https://github.com/maderix/ANE
- **Date:** Created 2026; last commit March 10, 2026
- **Summary:** Proof-of-concept that drives the Apple Neural Engine directly via the undocumented `_ANEClient`, `_ANECompiler`, and `_ANEInMemoryModelDescriptor` private APIs, bypassing CoreML entirely. It runs full transformer forward+backward passes on ANE hardware using IOSurface zero-copy tensor I/O in `[1, channels, 1, spatial]` layout, and includes a technique described as "ANE SRAM bandwidth probing." The repo extends the work described in the maderix Substack Part 3 (March 7, 2026) and is the most complete public reference for the `_ANEClient` API surface.
- **Why it matters:** The private API symbol set here (`_ANEClient`, `_ANECompiler`, `_ANEInMemoryModelDescriptor`) is the same surface any future ANE utilization hook in t3rm1nu55-monitorplus would need to call; the SRAM bandwidth probing technique may offer a utilization proxy beyond IOReport energy sampling.

### Finding 2: Orion — arXiv 2603.06728, ANE characterization for LLM training

- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 2026
- **Summary:** Orion delivers the first end-to-end compiler pipeline for transformer training and inference directly on the Apple Neural Engine, building on the maderix private-API approach. Key empirical discoveries: deep operation graphs (16–64 ops) achieve ~94% ANE utilization; the ANE compiler silently refuses compilations beyond ~119 per process (a hard per-process quota). The paper also confirms M4 Max ANE at 19 TFLOPS FP16 / 2.8 W under sustained load.
- **Why it matters:** The 119-compilation limit is a hard constraint for any monitoring tool that synthesizes probe workloads to infer ANE activity; the 94% utilization figure implies a characterization methodology worth understanding for designing utilization proxies.

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
