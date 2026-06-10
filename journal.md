# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-10 — sweep (5 findings)

### Finding 1: Orion — first end-to-end ANE compiler and runtime (arXiv 2603.06728)
- **Source:** arXiv (ANE search string: "Apple Neural Engine" AND "utilization")
- **URL:** https://arxiv.org/abs/2603.06728 · GitHub: https://github.com/mechramc/Orion
- **Date:** 2026-03-06
- **Summary:** Orion characterizes the M-series ANE and finds that deep operation graphs (16–64 ops) saturate the dispatch pipeline, reaching 94% of rated TOPS. It ships an open-source compiler that lowers a graph IR through five optimization passes to ANE-native MIL, plus a runtime using IOSurface-backed zero-copy tensor I/O and delta compilation for weight updates. No new hardware counter is exposed; utilization is inferred from wall-clock throughput benchmarks against known-peak TOPS. Companion GitHub: mechramc/Orion (MIT license).
- **Why it matters:** The deepest public ANE benchmark and compiler pipeline to date; the IOSurface buffer layout and `_ANEClient` invocation pattern are directly applicable to any future utilization-via-proxy approach in t3rm1nu55-monitorplus.

### Finding 2: maderix/ANE — open-source code for direct ANE execution and training
- **Source:** GitHub (maderix — tracked via substack reference)
- **URL:** https://github.com/maderix/ANE
- **Date:** 2026-03-05 (first commit); last active 2026-03-10
- **Summary:** Companion code to the maderix M4 ANE substack series (already tracked). Directly invokes `_ANEClient` / `_ANECompiler` private APIs and programs ANE kernels via IOSurface shared memory without CoreML. Benchmarks on M4: 35.1 TOPS FP16, 18.6 TOPS INT8 W8A8. Includes W&B power/loss logging during training steps — power is sampled via IOReport Energy Model alongside the training loop.
- **Why it matters:** First public reference implementation calling `_ANEClient` at the binary level; the IOSurface buffer format (`[1, channels, 1, spatial]`, fp16) and symbol names are directly usable for studying ANE activity detection patterns.

### Finding 3: clf3 blog — M3/M4 PMU ESR register format differs from M1/M2
- **Source:** blog.clf3.org (new source, not previously tracked)
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** unknown (published early 2026; corroborated by LKML Nick Chan v10 patchset, 2026-01-01)
- **Summary:** The author documents that M3 and M4 PMU ESR registers are 64-bit wide with each programmable event field taking 16 bits, versus the narrower format on M1/M2. This matches Nick Chan's upstream Linux kernel patch series ([PATCH v10 04/21] drivers/perf: apple_m1: support a per-implementation number of counters, https://lkml.org/lkml/2026/1/1/88), which adds chip-generation-aware counter configuration to the Apple PMU driver.
- **Why it matters:** kperf counter programming using M1/M2 ESR bit widths will silently misfire on M3/M4; the privileged sidecar needs chip-generation detection (via `sysctlbyname("hw.cpufamily")`) before writing event selectors.

### Finding 4: darwin-kperf — Rust crate bindings for kperf/kperfdata
- **Source:** crates.io (new source, not previously tracked)
- **URL:** https://crates.io/crates/darwin-kperf
- **Date:** Last updated 2026-02; companion `darwin-kperf-criterion` v0.1.0 released 2026-02-23
- **Summary:** Open-source Rust bindings for Apple's private `kperf.framework` and `kperfdata.framework`, authored by Bilal Mahmoud at HASH. Wraps the same undocumented kperf/kpc ABI used by ibireme's Objective-C gist. Includes a `darwin-kperf-criterion` crate for Criterion.rs benchmark integration. Requires root or the `com.apple.private.kernel.kpc` entitlement. No ANE/AMX-specific events exposed.
- **Why it matters:** Eliminates the need to write raw kperf FFI from scratch in the main project's privileged sidecar; Apache-2/MIT dual license. Evaluate as a dependency before hand-rolling kperf bindings.

### Finding 5: vladkens/macmon — active residency ratio IOReport channels
- **Source:** GitHub (vladkens/macmon — tracked)
- **URL:** https://github.com/vladkens/macmon/commit/3010f1fb7e209c334a1f948ea4386d92e1d761d2
- **Date:** 2026-06-09
- **Summary:** macmon now exposes `cpu_active_ratio`, `ecpu_active_ratio`, `pcpu_active_ratio`, and `gpu_active_ratio` as raw (non-frequency-scaled) active residency fractions from IOReport. GPU reads from the `"GPUPH"` channel within the `"GPU Stats"` IOReport group. The distinction: *active residency* = fraction of time the unit is not clock-gated; *effective usage* = frequency-scaled version of the same.
- **Why it matters:** The `"GPUPH"/"GPU Stats"` channel name and active-vs-effective residency pattern should be verified against t3rm1nu55-monitorplus's IOReport reader to ensure we expose both variants; upstream macmon is now ahead on this API surface.

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
