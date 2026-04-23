# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-04-23 — sweep (4 findings)

### Finding 1: Orion — first systematic ANE utilization characterization (arXiv 2603.06728)
- **Source:** arXiv / mechramc
- **URL:** https://arxiv.org/abs/2603.06728 | https://github.com/mechramc/Orion
- **Date:** March 2026 (arXiv submission 2603.06728; not captured in initial seed)
- **Summary:** Orion is the first open end-to-end system to drive the ANE directly via private `_ANEClient`/`_ANECompiler` APIs, bypassing CoreML entirely. The accompanying paper catalogs 20 behavioral constraints on MIL IR programs, 14 of which were previously undocumented, and measures ANE utilization as time-in-eval vs. wall time — reaching 94% with deep operation graphs (16–64 ops). Source available under MIT license.
- **Why it matters:** Closest public answer yet to "what does ANE utilization look like as a metric" — the time-in-eval proxy is the same approach t3rm1nu55-monitorplus would need to implement.

### Finding 2: macmon v0.7.0 — M5 IOReport `voltage-states` key rename breaks earlier parsers
- **Source:** vladkens/macmon
- **URL:** https://github.com/vladkens/macmon/releases/tag/v0.7.0
- **Date:** 2026-04-01 (not captured in initial seed)
- **Summary:** macmon v0.7.0 fixed a crash on M5 Max caused by IOReport renaming its `voltage-states` keys and added support for the new 3-core-type (E/P/S) label scheme introduced in M5 Pro/Max. The crash is a silent sentinel: any code that hard-codes `voltage-states` key names against M1–M4 conventions will crash or silently return zero on M5 Max.
- **Why it matters:** t3rm1nu55-monitorplus patterns IOReport access on macmon; this rename must be mirrored before shipping M5 support, or the kperf sidecar will see corrupt frequency residency data.

### Finding 3: M5 Pro/Max introduces "Super" (S) core type — 3-tier DVFS in IOReport
- **Source:** The Eclectic Light Company
- **URL:** https://eclecticlight.co/2026/04/13/cpu-core-frequencies-updated-for-all-current-apple-silicon-macs/
- **Date:** 2026-04-13
- **Summary:** M5 Pro and M5 Max replace E-cores with P-cores and add a new S (Super) class, giving a 3-tier core hierarchy for the first time. Idle frequencies for P/S cores doubled (~600 MHz → 1,308 MHz) and peak frequencies rose ~50% to 4,608 MHz. IOReport DVFS residency channels now carry 3 bins per cluster rather than 2.
- **Why it matters:** Any hardcoded 2-tier (P/E) DVFS residency parsing will silently miscount or panic on M5 Pro/Max; the S-core bin needs a third parse branch.

### Finding 4: lambdafoo `mperf` — practical kperf/kperfdata guide with kpep file map
- **Source:** Perpetually Curious Blog (lambdafoo.com)
- **URL:** https://lambdafoo.com/posts/2026-03-25-mperf-hardware-counters-macos.html
- **Date:** 2026-03-25 (not captured in initial seed)
- **Summary:** Introduces `mperf`, a `perf stat`-like CLI for Apple Silicon that drives `kperf.framework` and `kperfdata.framework` entirely from userspace via `dlsym`. Confirms chip-specific kpep event databases at `/usr/share/kpep/` (a14.plist → M1, a15.plist → M2, as4.plist → M4) and documents the Profile-Every-Thread (PET) kernel timer mechanism. No ANE/AMX-specific events found in the M4 kpep database.
- **Why it matters:** Confirms M4 kpep filename (`as4.plist`) and PET mechanism details needed for the kperf privileged sidecar design; the absence of ANE events in as4.plist is itself a confirmed negative data point.

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
