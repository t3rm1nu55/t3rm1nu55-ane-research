# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-31 — sweep (5 findings)

### Finding 1: Bryngelson — "Apple Neural Engine: Architecture, Programming, and Performance"
- **Source:** arXiv / ane-guide.readthedocs.io
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 2026
- **Summary:** Spencer Bryngelson's paper provides the first complete reverse-engineered account of the ANE from silicon datapath to system interface, derived from direct measurement on Apple Silicon and static decompilation of the private runtime, compiler, kernel driver, and firmware. It covers the datapath, memory hierarchy, per-chip rooflines, weight-compression scheme, kernel driver, firmware, and command protocol. A companion web reference is live at https://ane-guide.readthedocs.io and a GitHub repo `sbryngelson/ane-guide` holds the source.
- **Why it matters:** Now the authoritative public reference for ANE internals below CoreML; the kernel-driver and firmware-protocol documentation is the prerequisite for determining whether hardware counters exist and how they could be addressed.

### Finding 2: ANEForge — Python library for direct ANE dispatch
- **Source:** arXiv + GitHub (sbryngelson/ANEForge)
- **URL:** https://arxiv.org/abs/2606.17090 / https://github.com/sbryngelson/ANEForge
- **Date:** June 12, 2026
- **Summary:** ANEForge is an open-source Python library (available on PyPI) that compiles a lazy tensor graph of 58 fused operators directly into a single ANE program and dispatches it through the same ANE daemon and kernel-driver stack used by Apple's private framework — completely bypassing CoreML. It supports inference, training (forward + backward + Adam), LLM decode/prefill with Llama/Qwen, and scientific computing kernels.
- **Why it matters:** Provides an open, inspectable implementation of the full ANE dispatch path — exactly the code path any utilization-measurement approach would need to instrument or hook; useful as a reference for what observable state exists during an ANE dispatch.

### Finding 3: Orion — first open ANE runtime for LLM training
- **Source:** arXiv + GitHub (mechramc/Orion)
- **URL:** https://arxiv.org/abs/2603.06728 / https://github.com/mechramc/Orion
- **Date:** March 6, 2026
- **Summary:** Orion is a native runtime that trains and runs small LLMs directly on ANE, bypassing CoreML entirely via the private `_ANEClient` and `_ANECompiler` APIs, and is the first open system to combine direct ANE execution, a compiler pipeline, and stable multi-step training with checkpoint resume.
- **Why it matters:** A third independent implementation (alongside maderix/ANE and ANEForge) confirming the `_ANEClient` access pattern is stable and reproducible; the ecosystem around direct ANE access has now matured enough to anchor a counter-exposure investigation.

### Finding 4: AMX→SME architectural break confirmed for M4+
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" (Deyvik Bhan) explicitly confirms that M1–M3 use Apple's proprietary AMX coprocessor while M4 and later switch to ARM's standard Scalable Matrix Extension (SME). Microbenchmarks show M1 AMX is load-issue bound at ~610–680 GFLOPS under realistic mixed-load conditions.
- **Why it matters:** t3rm1nu55-monitorplus must handle M4+ differently for any matrix-unit utilization tracking — SME is a standard ARM extension with documented PMU events, unlike the undocumented Apple AMX, which opens a new counter access path on M4+.

### Finding 5: LKML Nick Chan v10 — Apple A7-A11/T2 PMU patchset
- **Source:** LKML
- **URL:** https://lkml.org/lkml/2026/1/1/82
- **Date:** January 1, 2026
- **Summary:** Nick Chan posted v10 of a 21-patch series expanding the Linux `apple_m1` PMU driver to cover Apple A7-A11 SoCs and the T2 coprocessor, including DT bindings and per-implementation PMU startup support. v10 indicates high maturity and likely imminent upstream acceptance.
- **Why it matters:** Progress on the Linux Apple PMU driver deepens the open-source understanding of Apple Silicon counter architecture; the `apple_m1` driver is the closest open-source reference for M-series PMU counter semantics and its upstream acceptance improves long-term visibility.

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
