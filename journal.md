# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-07 — sweep (3 findings)

### Finding 1: maderix/ANE — open-source code for direct ANE access via private APIs
- **Source:** github.com/maderix/ANE (newly discovered, not previously tracked)
- **URL:** https://github.com/maderix/ANE
- **Date:** Created ~March 2026; 6.7k stars as of June 2026
- **Summary:** Companion code repo to the maderix M4 ANE substack series. Implements full transformer training (forward + backward + Adam) directly on ANE hardware by reverse-engineering `_ANEClient`, `_ANECompiler`, and 40+ private `AppleNeuralEngine.framework` symbols including `_ANEInMemoryModel` and `_ANEIOSurfaceObject`. Achieves ~9.3 ms/step on Stories110M at 11.2% ANE utilization (≈1.78 TFLOPS sustained); the low utilization is attributed to element-wise ops still falling back to CPU. The IOKit driver interaction layer is fully documented in the repo.
- **Why it matters:** The IOKit call sequence in maderix/ANE is the most complete public model of ANE driver interaction; studying it could reveal whether any power/utilization registers are readable from the host side during ANE execution.

### Finding 2: ClF3 blog — M3/M4 PMU ESR register format diverges from M1/M2
- **Source:** blog.clf3.org (newly discovered, not previously tracked)
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** Published 2025–2026 (exact date unavailable; not indexed with date)
- **Summary:** Documents a breaking PMU register format change introduced in M3: the `SYS_APL_PMEVSR*_EL1` event-selection registers on M3/M4 use **16-bit event fields** per slot, vs. 8-bit on M1/M2 — the encoding is not backward-compatible. Also notes that writes to `SYS_APL_PMCR0_EL1` are reverted by the kernel within ~100 µs unless the OS is explicitly patched to allow persistent counter configuration.
- **Why it matters:** The monitorplus kperf sidecar currently targets M1/M2 event encoding. M3 and M4 hosts will silently misconfigure counters unless the ESR field width is branched per chip generation — this is a concrete bug risk for any user on M3/M4 hardware.

### Finding 3: arxiv 2604.18788 — NPUMoE: MoE inference runtime targeting Apple Silicon NPU
- **Source:** arXiv (search string: "Apple Neural Engine" AND "benchmark")
- **URL:** https://arxiv.org/abs/2604.18788
- **Date:** April 20, 2026
- **Summary:** Presents NPUMoE, a runtime that offloads static, dense expert kernels in MoE LLMs to the Apple Silicon NPU (ANE) while routing dynamic operations to CPU/GPU. Uses offline calibration to predict expert capacity and popularity, enabling static ANE compute graphs. Reports 1.32–5.55× latency reduction and 1.81–7.37× energy efficiency improvement vs. CPU/GPU baselines across M-series devices.
- **Why it matters:** No new counter APIs exposed, but NPUMoE's "offline calibration → static graph" approach is the clearest published methodology for characterizing ANE throughput in production workloads; energy efficiency figures are derived from wall-clock + IOReport energy deltas, confirming IOReport Energy Model remains the practical utilization proxy.

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
