# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-09 — sweep (4 findings)

### Finding 1: Orion — first open ANE runtime with measured 94% utilization
- **Source:** arXiv 2603.06728 + github.com/mechramc/Orion
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026
- **Summary:** Orion is the first complete open system that bypasses CoreML entirely via `_ANEClient`/`_ANECompiler`, compiles Apple MIL programs to E5 microcode, and reports 94% ANE utilization on deep operation graphs (16–64 ops). The paper catalogs 20 ANE constraints (14 previously undocumented), and the runtime handles IOSurface-backed fp16 tensor I/O with delta-compilation — surgical weight updates via unload/reload without full recompilation. Full Objective-C + Python source is MIT-licensed at github.com/mechramc/Orion.
- **Why it matters:** Orion's measured 94% utilization threshold is the first published calibration point for ANE load estimation; the MIT-licensed `_ANEClient` wrapper is a direct model for a Rust FFI hook in t3rm1nu55-monitorplus.

### Finding 2: maderix/ANE GitHub repo — C bridge, SRAM bench, training utilization data
- **Source:** github.com/maderix/ANE + maderix Substack Part 3
- **URL:** https://github.com/maderix/ANE
- **Date:** March 2026 (Part 3 published March 7, 2026)
- **Summary:** The maderix ANE project matured from blog posts into an active GitHub repo (42 commits). It provides `ane_bridge.m/h` (C-callable wrapper), `sram_bench.m` (SRAM bandwidth and layout characterization), and `ane_int8_bench.m` (INT8 vs FP16 throughput comparisons). Measured training utilization is ~5-9% of peak; M4 peak is 18.6 TFLOPS FP16, 35.1 TFLOPS INT8 W8A8. M5 throughput data has been contributed by community member m0at.
- **Why it matters:** `ane_bridge.m/h` is a C-callable shim that could be wrapped by Rust `bindgen` to expose ANE dispatch counts and training utilization rates to t3rm1nu55-monitorplus without writing new Objective-C.

### Finding 3: ClF3 blog — M3/M4 PMU ESR registers are 64-bit with 16-bit event selectors
- **Source:** ClF3's blog (blog.clf3.org)
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** Unknown (discovered this sweep; not previously in references)
- **Summary:** ClF3 documents that the key architectural difference between M1/M2 and M3/M4 PMU is that M3/M4 ESR (Event Selection Register) fields are 64-bit with 16 bits allocated per event, whereas M1/M2 use 8 bits per event. This is a hard incompatibility: kperf event codes that work on M1/M2 are not byte-compatible with M3/M4 register layouts.
- **Why it matters:** t3rm1nu55-monitorplus's kperf sidecar currently hardcodes M1/M2 event selection logic; it will silently misread counters on M3/M4 hardware without this fix. Directly actionable.

### Finding 4: NPUMoE paper — ANE throughput benchmarks on M2 and M3 (April 2026)
- **Source:** arXiv 2604.18788
- **URL:** https://arxiv.org/abs/2604.18788
- **Date:** April 20, 2026
- **Summary:** NPUMoE demonstrates MoE-style LLM inference dispatched directly to Apple Silicon ANE, achieving 1.32x–5.55x latency reduction and 1.81x–7.37x energy efficiency improvement over baselines. The paper benchmarks a 16-core M2 ANE at up to 15.8 TFLOPS FP16. Measurement methodology is black-box throughput benchmarking, not hardware counter access.
- **Why it matters:** Provides updated, reproducible ANE throughput figures for M2/M3 that can serve as calibration ground-truth when indirect utilization estimation is implemented.

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
