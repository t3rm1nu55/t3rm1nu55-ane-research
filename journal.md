# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-24 — sweep (4 findings)

### Finding 1: vladkens/macmon now exposes per-cluster active residency ratios via IOReport
- **Source:** vladkens/macmon (GitHub)
- **URL:** https://github.com/vladkens/macmon/commit/3010f1fb7e209c334a1f948ea4386d92e1d761d2
- **Date:** June 9, 2026
- **Summary:** macmon added a feature exposing "active residency ratios" from IOReport — the fraction of time each CPU/GPU cluster spent in an active P-state per sample interval. The same `feat: add fan speed metrics` batch also added fan RPM from IOReport. Both features shipped in what is heading toward v0.8.
- **Why it matters:** Active residency ratios are the best available per-cluster utilization proxy without hardware counters; t3rm1nu55-monitorplus should expose this IOReport channel alongside existing power metrics. **(Actionable — see issue)**

### Finding 2: mechramc/Orion — open-source ANE runtime, bypasses CoreML, documents 14+ undocumented constraints
- **Source:** mechramc/Orion (GitHub — new untracked repo; companion paper arxiv:2603.06728)
- **URL:** https://github.com/mechramc/Orion
- **Date:** March 2026 (not captured in April 7 initial seed; repo last updated June 22, 2026)
- **Summary:** Orion is an Objective-C runtime that bypasses CoreML entirely, compiling and executing LLMs directly on the ANE via `_ANEClient`/`_ANECompiler`. Key discoveries: (1) a "delta compilation" trick patches the weight BLOBFILE on disk without recompiling, bypassing the ~119-compile ANE session limit for an 8.5× training speedup; (2) multi-output IOSurface buffers are ordered alphabetically by MIL variable name, not by return-tuple position — a silent correctness trap; (3) minimum ANE IOSurface allocation is ~49KB. The accompanying arXiv paper catalogs 20 ANE constraints, 14 newly documented.
- **Why it matters:** Orion is now the deepest public characterization of the ANE private API surface and the most complete reference for anyone attempting ANE dispatch hooking or utilization inference via `_ANEClient`.

### Finding 3: maderix/ANE repo — macOS 26 breaks `compileModelAtURL`; M5 confirmed H16G ANE
- **Source:** maderix/ANE (GitHub — new untracked repo, companion to tracked Substack series)
- **URL:** https://github.com/maderix/ANE (key commit: https://github.com/maderix/ANE/commit/44309b76258e6b362d8fd075fec19041a17cf685)
- **Date:** March 2026 (not captured in April 7 initial seed)
- **Summary:** The open-source companion to the maderix Substack series documents that macOS 26 breaks `compileModelAtURL` in the private ANE compiler; the fix is switching to in-memory MIL compilation. Community-submitted cross-generation benchmarks confirm M5 uses the same H16G ANE core architecture as M4 (16 cores, same SRAM cliff), with INT8 W8A8 measuring 35.1 TOPS on M4 vs. the marketed "38 TOPS" figure.
- **Why it matters:** The macOS 26 API break is a direct hazard for any code path using file-based `_ANECompiler` compilation — must use in-memory MIL on macOS 26+. M5 sharing the H16G architecture means existing M4 IOReport energy channel mappings should carry forward.

### Finding 4: clf3 blog — new practical guide to PMU event counters on M3/M4
- **Source:** clf3's blog (new untracked source, found via search)
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** Unknown (URL inaccessible in this sweep due to proxy 403; source confirmed via web search)
- **Summary:** A blog post at blog.clf3.org describes a practical approach to accessing PMU event counters on Apple M3 and M4 processors via the kperf private API. Content could not be directly fetched this sweep; described in secondary sources as covering counter group constraints and M3/M4-specific nuances.
- **Why it matters:** New untracked source for kperf counter access; should be manually fetched and reviewed for any M3/M4-specific event IDs or constraint rules that update the bugsiki reference already tracked.

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
