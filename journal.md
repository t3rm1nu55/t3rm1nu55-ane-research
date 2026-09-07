# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-09-07 — sweep (6 findings)

### Finding 1: "Apple Neural Engine: Architecture, Programming, and Performance" — comprehensive reverse-engineering reference
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 21 2026 (v1); August 18 2026 (v2)
- **Summary:** Spencer Bryngelson reverse-engineered the full ANE software stack — datapath roofline, dispatch route below CoreML, compiler and on-disk program format, weight-compression scheme, kernel driver, firmware, and command protocol — with direct measurements on M1 and M5. A companion web edition and GitHub guide (`sbryngelson/ane-guide`) are maintained as a living reference.
- **Why it matters:** Now the single most authoritative public reference for ANE internals; informs where counters or telemetry hooks might exist in the driver/firmware layer.

### Finding 2: ANEForge — Python package for direct ANE dispatch
- **Source:** arXiv + GitHub + PyPI
- **URL:** https://arxiv.org/abs/2606.17090 / https://github.com/sbryngelson/ANEForge
- **Date:** June 2026
- **Summary:** ANEForge is a Python library (on PyPI as `aneforge`) that compiles a lazy tensor graph using 58 fused operators into a native ANE program and dispatches it through `_ANEClient` and the ANE kernel driver, bypassing CoreML entirely. It supports inference and training including backward pass and optimizer update, reaching 18.6 TFLOPS FP16 on M4.
- **Why it matters:** Demonstrates the full private dispatch stack is now well-understood; the dispatch path is the most likely location for any IOKit-level power or throughput telemetry hooks.

### Finding 3: ANE memory-controller byte counters confirmed readable
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2608.22110
- **Date:** August 22 2026
- **Summary:** "What actually runs: a measurement study of language model placement and decode speed on the Apple Neural Engine" reads the ANE's memory-controller byte counters during inference to confirm which operations actually executed on the engine vs. CPU or GPU. Their methodology distinguishes fused vs. decomposed operations that are arithmetically identical but differently ANE-eligible.
- **Why it matters:** First public confirmation that ANE memory-controller byte counters are readable and give empirical "was the ANE used" signal — the closest thing to a utilization counter the public record has documented so far.

### Finding 4: M3/M4 PMU ESR registers are 64-bit with 16-bit-per-event encoding
- **Source:** blog.clf3.org
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** 2026 (exact date not shown)
- **Summary:** Documents that on M3 and M4 the PMU ESR registers are 64-bit with each event taking 16 bits (vs. 8 bits on M1/M2), and that PMU counters themselves are 64-bit with bit 63 triggering PMI on M2+. Also notes `SYS_APL_PMCR0_EL1` is overwritten by a kernel process approximately every 100 µs, severely limiting time-budgeted counter access.
- **Why it matters:** The kperf sidecar in t3rm1nu55-monitorplus needs M3/M4-aware ESR parsing; the 100 µs kernel overwrite is a hard constraint the sampling loop must respect.

### Finding 5: maderix/ANE — transformer training on ANE via private APIs (new GitHub repo)
- **Source:** GitHub
- **URL:** https://github.com/maderix/ANE
- **Date:** 2026 (after April 2026 Substack post)
- **Summary:** maderix published a new GitHub repo extending the Substack reverse-engineering series into runnable code: transformer training (forward + backward + optimizer) on M4 ANE via `_ANEClient`, achieving 18.6 TFLOPS FP16 and 35.1 TFLOPS INT8, but reporting only 5–9% utilization of peak during training, with significant dispatch-overhead and compilation constraints remaining.
- **Why it matters:** The low utilization figure (5–9%) and documented dispatch overhead (~0.095 ms) give concrete numbers for the cost of non-batched IOKit dispatch — relevant to design of any sampling or benchmarking path in the main project.

### Finding 6: Nick Chan LKML patchset v10 — Apple PMU driver per-implementation event tables
- **Source:** LKML
- **URL:** https://lkml.iu.edu/2601.0/00044.html
- **Date:** January 2026 (v10 of an ongoing series)
- **Summary:** Nick Chan's 21-patch series extends the Linux `apple_m1` PMU driver with per-implementation event tables and per-implementation PMU startup, adding Apple A7–A11 support and addressing conflicts with FEAT_PMUv3 KVM support. This represents the most recent open effort to generalize Apple Silicon PMU support across generations in the Linux kernel.
- **Why it matters:** Event-table coverage in the Linux driver is a useful cross-reference for which events exist on which chips — any event added here that isn't in dougallj/applecpu is worth examining for the main project.

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
