# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-22 — sweep (4 findings)

Note: GitHub MCP tool access is restricted to this repo; GitHub repo checks were performed via web search. Findings 1–4 were published in March 2026 but missed by the initial seed (which was created the same day it was committed, before any sweep had run). All are new to the journal.

### Finding 1: maderix ANE trilogy complete — code repo published (maderix/ANE)
- **Source:** maderix Substack / GitHub
- **URL:** https://github.com/maderix/ANE · https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b (Part 3)
- **Date:** March 2026
- **Summary:** maderix completed the three-part M4 ANE reverse-engineering series and published all code at `github.com/maderix/ANE` (6 700+ stars). Part 3 demonstrates full transformer training (109 M params) on the ANE via direct IOKit driver access, mapping 40+ private classes from `_ANEClient`/`_ANECompiler`. Part 3 reports "ANE utilization at 11.2%" — a computed metric (actual TFLOPS / theoretical peak 15.8 TFLOPS), not a hardware counter readout. Code was tested on M4 Mac Mini, macOS 15.x.
- **Why it matters:** The `maderix/ANE` codebase is now the most complete public reference for direct ANE IOKit driver programming; its class mapping is the best available public baseline for what the ANE kernel interface exposes.

### Finding 2: Orion — first open ANE runtime, 20-constraint MIL catalog (arXiv:2603.06728)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** 6 March 2026
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" (Kumaresan) catalogs 20 constraints on ANE MIL IR programs, of which 14 are previously undocumented, and builds the first open end-to-end ANE runtime bypassing CoreML entirely via `_ANEClient`/`_ANECompiler`. It achieves 170+ tokens/s GPT-2 inference on M4 Max and discovers that the ANE compiler limits each process to ~119 compilations before silent failure.
- **Why it matters:** The constraint catalog and compiler-state limit are directly relevant to understanding the ANE's observable envelope from the host side; these bounds constrain what a future ANE utilization API could surface.

### Finding 3: jiegec/apple-pmu — rendered kpep event tables for M1–M4
- **Source:** GitHub — jiegec/apple-pmu
- **URL:** https://github.com/jiegec/apple-pmu
- **Date:** Present in searches as of this sweep (no single commit date identified)
- **Summary:** `jiegec/apple-pmu` dumps and renders Apple's PMU counter definitions from `/usr/share/kpep/` into human-readable markdown tables per chip generation (a14=M1, a15=M2, as3=M3, as4=M4). This makes it trivial to search all named kperf events on M4 (as4.md) for any AMX-adjacent or vector-unit event names.
- **Why it matters:** Directly complements `dougallj/applecpu` for kperf event enumeration; scanning `as4.md` for hidden AMX-adjacent counter names is now a one-step grep rather than a binary reverse-engineering exercise.

### Finding 4: lambdafoo — practitioner guide to kperf/kperfdata on macOS ARM64
- **Source:** lambdafoo.com blog
- **URL:** https://lambdafoo.com/posts/2026-03-25-mperf-hardware-counters-macos.html
- **Date:** 25 March 2026
- **Summary:** A practitioner guide to reading hardware performance counters on macOS ARM64 via the private `kperf.framework` / `kperfdata.framework` stack, including the `mperf` library which provides portable aliases (cycles, instructions, branch-misses, l1d-cache-misses) that resolve to the correct kpep event names per chip at runtime. Confirms: 2 fixed counters (cycles, instructions) + 8 configurable counters = 10 maximum simultaneous events on Apple Silicon.
- **Why it matters:** Closest available working reference implementation for the kperf FFI layer the monitorplus privileged sidecar needs; the 10-counter-slot ceiling is confirmed current on M4, constraining how many PMU events can be multiplexed simultaneously.

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
