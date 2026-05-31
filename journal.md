# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-31 — sweep (3 findings)

### Finding 1: clf3.org — M3/M4 PMU counters are 64-bit with 16-bit ESR event encoding
- **Source:** clf3.org technical blog
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** Unknown (discovered this sweep; not in prior journal)
- **Summary:** Documents that M3 and M4 performance counters are 64-bit registers, versus M1's 48-bit, and that the ESR_EL1 format on M3/M4 allocates 16 bits per event field rather than fewer. The post walks through low-level register reads directly from Asahi Linux kernel code and extends beyond what bugsiki.dev covers for M1/M2.
- **Why it matters:** The kperf sidecar in t3rm1nu55-monitorplus must handle counter-width differences per chip generation; any M3/M4 counter-reading code that assumes 48-bit saturation or older ESR bit-field layouts will silently produce wrong values.

### Finding 2: Orion — first open system to run on ANE via private APIs without CoreML
- **Source:** arXiv 2603.06728 + github.com/mechramc/Orion
- **URL:** https://arxiv.org/abs/2603.06728 / https://github.com/mechramc/Orion
- **Date:** March 4–6, 2026 (not captured in April 7 seed)
- **Summary:** Orion is the first open end-to-end runtime that bypasses CoreML entirely via Apple's private `_ANEClient` and `_ANECompiler` APIs, enabling both inference and training directly on the ANE. Key empirical discoveries: 16–64-op graphs achieve 94% ANE utilization; a hard limit of ~119 `_ANECompiler` invocations per process exists before the compiler daemon refuses further compilations, requiring a process-restart workaround.
- **Why it matters:** Confirms and systematizes the private API surface the project may eventually use for ANE dispatch-rate inference; the 119-compile-per-process limit is a hard constraint any long-running monitor must account for.

### Finding 3: maderix Part 3 + maderix/ANE repo — transformer training on ANE, new public code
- **Source:** maderix Substack (tracked) + github.com/maderix/ANE (new repo)
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b / https://github.com/maderix/ANE
- **Date:** March 7, 2026 (Part 3 not captured in April 7 seed; GitHub repo not previously tracked)
- **Summary:** Part 3 demonstrates full transformer training (forward pass, backward pass, Adam optimizer) on 109M–596M parameter models directly on the M4 ANE via reverse-engineered private APIs, with a public MIT-licensed code repository. It does not expose new hardware counters but uses the same `_ANEClient` private API surface documented in the Orion paper.
- **Why it matters:** Adds a reference implementation to track for private ANE API usage patterns; the maderix/ANE repo should be added to references.md as a code-level complement to the Substack writeups.

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
