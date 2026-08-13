# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-13 — sweep (5 findings)

### Finding 1: Apple Neural Engine Architecture, Programming, and Performance — comprehensive reverse-engineered reference
- **Source:** arXiv + ane-guide.readthedocs.io (Spencer H. Bryngelson, Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.22283 — web edition: https://ane-guide.readthedocs.io
- **Date:** June 21, 2026 (v2: June 27, 2026)
- **Summary:** Most comprehensive public documentation of the ANE to date, covering the full stack: datapath and roofline, dispatch route below CoreML, compiler pipeline, on-disk e5rt program format, weight-compression scheme, kernel driver, firmware, and command protocol. Based on direct measurement and static analysis of private runtime symbols on real Apple Silicon hardware.
- **Why it matters:** New authoritative reference for ANE internals — directly informs what `_ANEClient`/`_ANECompiler` expose and how a utilization hook could be seated below CoreML in t3rm1nu55-monitorplus.

### Finding 2: ANEForge — Python frontend for direct ANE computation without CoreML
- **Source:** arXiv / GitHub sbryngelson/ANEForge / PyPI `aneforge`
- **URL:** https://arxiv.org/abs/2606.17090 — https://github.com/sbryngelson/ANEForge
- **Date:** June 12, 2026
- **Summary:** Same author as Finding 1. Compiles Python tensor graphs into single fused e5rt ANE programs via private `_ANEClient`/`_ANECompiler` APIs, dispatching through the ANE daemon and kernel-driver stack without CoreML. Catalogs 58 fused operators and 19 native bridge operators; ships as a PyPI package.
- **Why it matters:** Most complete public catalog of the ANE's operator surface to date; the dispatch path it maps is the most plausible location for inserting a utilization measurement hook.

### Finding 3: "Above the Inner Loop" — M1 has two on-chip AMX blocks; inner loop is load-issue bound
- **Source:** arXiv (Deyvik Bhan, Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Shows that the M1 carries two on-chip AMX blocks and that the single-thread FMA32 inner loop is load-issue bound at ~610–680 GFLOPS (under half the load-free theoretical rate). Multi-thread panel sizing to fill both blocks recovers throughput. Characterizes the M1–M3 AMX microarchitecture as sharing this topology.
- **Why it matters:** First public confirmation of dual AMX block topology on M1–M3; no new counters, but sets the microarchitecture baseline needed to reason about any future AMX utilization metric.

### Finding 4: clf3 blog — M3/M4 PMU registers differ from M1/M2; PMCR0_EL1 is kernel-clobbered every ~100µs
- **Source:** blog.clf3.org
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** 2026 (exact date not retrieved; proxy-blocked)
- **Summary:** Documents that on M3/M4, PMU ESR registers are 64-bit with each event occupying 16 bits (different from M1/M2 layout), and bit 63 triggers the performance monitoring interrupt. The critical finding: `SYS_APL_PMCR0_EL1` is repeatedly overwritten by a kernel process — modifications do not persist longer than ~100µs.
- **Why it matters:** Direct blocker for the kperf privileged sidecar on M3/M4 hardware: counter-enable state must be re-armed at sub-100µs cadence or an alternative enablement path found. Requires design changes in the sidecar before M3/M4 support can land.

### Finding 5: SiliconScope v4.0 — shipping sudoless ANE utilization display with remote monitoring
- **Source:** GitHub kennss/SiliconScope
- **URL:** https://github.com/kennss/SiliconScope
- **Date:** Active in 2026; v4.0 is current
- **Summary:** Native SwiftUI monitor providing sudoless ANE, Media Engine, and memory-bandwidth tracking alongside CPU/GPU/power/sensors. v4.0 added remote Mac monitoring including Neural Engine metrics. Signed and notarized; requires macOS 14+ on Apple Silicon.
- **Why it matters:** Proof that sudoless ANE utilization display is achievable in a shipping app. The IOReport channels or estimation approach used by SiliconScope are worth reverse-engineering as an alternative to the power-indirection method currently planned for t3rm1nu55-monitorplus.

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
