# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-20 — sweep (4 findings)

### Finding 1: maderix/ANE — open-source direct ANE access with utilization measurement
- **Source:** github.com/maderix/ANE + maderix Substack Part 3
- **URL:** https://github.com/maderix/ANE
- **Date:** March 2, 2026 (not captured in initial seed)
- **Summary:** maderix published a working implementation of transformer training directly on the M4 ANE via reverse-engineered `_ANEClient`, `_ANECompiler`, and `_ANEInMemoryModelDescriptor` private APIs, bypassing CoreML entirely. The project measures ANE utilization efficiency: 11.2% at a single-layer workload (1.78 TFLOPS), scaling to 94% at 32+ chained operations approaching the 19 TFLOPS ceiling. Part 3 of the Substack series demonstrates a full forward+backward pass and Adam optimizer updates running natively on ANE hardware.
- **Why it matters:** The utilization efficiency metric demonstrates that ANE throughput relative to peak is observable — the source of this metric (API timing, power-ratio, or internal counter) needs investigation as a potential foundation for the monitorplus ANE utilization signal.

### Finding 2: Orion — first open end-to-end ANE characterization system (arXiv 2603.06728)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" is the first open system bypassing CoreML via `_ANEClient`/`_ANECompiler`; it catalogs 20 ANE MIL IR constraints (14 previously undocumented) and achieves 170+ tokens/s for GPT-2 124M inference on M4 Max. A weight-patching technique reduces per-step recompilation from 4,200 ms to 494 ms (8.5x speedup), enabling viable training loops.
- **Why it matters:** The most rigorous public characterization of ANE throughput to date; the constraint catalog and compiler pipeline are the closest public analogue to the infrastructure needed to inject real-time monitoring probes into the ANE software stack.

### Finding 3: clf3.org — PMU event counters documented on M3 and M4
- **Source:** clf3.org blog
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** Unknown (discovered this sweep; not in initial seed)
- **Summary:** Post extending kperf/kpc PMU event counter documentation specifically to M3 and M4 chips. Key structural difference vs M1/M2: M3/M4 ESR registers are 64-bit with 16 bits per event. Covers the kpep database format and counter-group compatibility rules for these newer chips.
- **Why it matters:** The only public source extending the bugsiki.dev counter reference to M3/M4; any AMX or ANE-adjacent events appearing uniquely in M3/M4 counter tables would be directly actionable for the kperf sidecar in monitorplus.

### Finding 4: macmon v0.7.2 — M5 chip support confirmed, ANE power channel intact
- **Source:** vladkens/macmon
- **URL:** https://github.com/vladkens/macmon/releases
- **Date:** May 2, 2026
- **Summary:** macmon v0.7.2 extends support to M5, advertising "M1–M5" in its description, with ANE power still reported as a separate IOReport Energy Model metric. No change to the IOReport channel access pattern was required for M5 compatibility.
- **Why it matters:** Confirms the IOReport power-indirection approach for ANE activity detection works on M5 without modification — the existing monitorplus channel access pattern needs no changes for the newest Apple Silicon generation.

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
