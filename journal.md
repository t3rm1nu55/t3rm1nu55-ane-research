# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-04-16 — sweep (3 findings)

### Finding 1: Orion — first open-source ANE runtime with wall-time utilization metric
- **Source:** arXiv + github.com/mechramc/Orion
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 2026
- **Summary:** Orion is the first open system to bypass CoreML entirely and program the ANE directly via the `_ANEClient` and `_ANECompiler` private APIs, supporting both inference and full transformer training. The paper catalogs 20 ANE execution constraints (14 newly discovered MIL IR, memory, and I/O constraints) and the runtime exposes a concrete utilization signal: wall time spent inside `orion_eval` vs total wall time. Softmax over large vocabularies benchmarks at 33.8× faster on ANE than CPU.
- **Why it matters:** Closest thing to an ANE "utilization counter" yet published; the constraint catalog and `orion_eval` timing approach are directly applicable to t3rm1nu55-monitorplus ANE telemetry planning.

### Finding 2: maderix Part 3 — full transformer training on M4 ANE, 119-compile cap documented
- **Source:** maderix substack
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** March 7, 2026
- **Summary:** Part 3 of the tracked maderix M4 ANE series demonstrates full forward+backward pass training of a 109M-parameter transformer (scaled to Qwen3-0.6B) using only `_ANEClient`/`_ANECompiler`. It documents an undocumented `_ANEClient` constraint: each process is limited to ~119 compilations, after which subsequent compilations silently fail; the fix is an exec()-restart costing ~50 ms/step, or delta compilation which eliminates the cap entirely.
- **Why it matters:** The 119-compile cap is a critical `_ANEClient` operational constraint that would affect any monitor process dynamically interacting with the ANE over time.

### Finding 3: Apple M5 ships (March 2026) — M5 ANE and kperf event surface uncharted
- **Source:** Apple newsroom
- **URL:** https://www.apple.com/newsroom/2026/03/apple-introduces-the-new-macbook-air-with-m5/
- **Date:** March 2026
- **Summary:** Apple shipped the MacBook Air with M5 in March 2026, introducing a new ANE generation with up to 4× advertised AI performance over M4. No community dump of the M5 kpep event database has appeared yet, and the bugsiki PMU analysis (the reference source for counter slot allocation) does not extend beyond M4.
- **Why it matters:** t3rm1nu55-monitorplus will need M5 PMU event mappings before supporting M5 hardware; the first community kpep dump for M5 is a near-term research milestone to watch.

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
