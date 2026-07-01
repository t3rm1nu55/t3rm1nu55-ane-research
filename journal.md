# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-01 — sweep (5 findings)

### Finding 1: arXiv:2606.22283 — "Apple Neural Engine: Architecture, Programming, and Performance"
- **Source:** arXiv (new paper, not previously tracked)
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 21, 2026
- **Summary:** Spencer H. Bryngelson's reverse-engineered account of ANE internals spanning A11–A18 and M1–M5, based on static analysis of the private runtime, compiler, kernel driver, firmware, and direct measurement on M1 and M5 hardware. Documents the full dispatch route below CoreML, the on-disk program/weight-compression format, and the command protocol exchanged between driver and firmware. Per-chip target tables and an operation-by-device matrix are included.
- **Why it matters:** The kernel driver and command protocol documentation is the closest thing yet to a public roadmap for where ANE hardware counters would be exposed — directly relevant if ANE counter access ever emerges.

### Finding 2: arXiv:2604.18788 — "Efficient Mixture-of-Experts LLM Inference with Apple Silicon NPUs"
- **Source:** arXiv (new paper, not previously tracked)
- **URL:** https://arxiv.org/abs/2604.18788
- **Date:** April 20, 2026
- **Summary:** Introduces NPUMoE, a runtime that accelerates MoE LLM inference on ANE by offloading dense, static computation while preserving CPU/GPU fallback for dynamic ops. Documents ANE's concurrency limits, shape constraints, and the overhead of expert dispatch — shows ANE prefill latency dominates (80% of end-to-end) under current dispatch model.
- **Why it matters:** Quantifies ANE dispatch overhead and concurrency limits at the workload level, which is useful context for any future per-inference utilization metric.

### Finding 3: github.com/maderix/ANE — MIT-licensed direct ANE execution repo
- **Source:** GitHub (new repo discovered from maderix blog series, Part 3)
- **URL:** https://github.com/maderix/ANE
- **Date:** March 2026 (active; Part 3 blog post published March 2026)
- **Summary:** Open-source MIT-licensed implementation of direct ANE training (forward + backward pass) via reverse-engineered `_ANEClient`/`_ANECompiler` private APIs. Trained 109M-parameter transformer and scaled to Qwen3-0.6B on M4 ANE. Includes SRAM exploration and bandwidth characterization tooling. The blog series (Part 1–3 on maderix Substack) documents 19 TFLOPS FP16 @ 2.8W, a 32 MB SRAM cliff, ~0.095 ms dispatch overhead, and a ~119 compile/process limit.
- **Why it matters:** First open implementation showing ANE dispatch mechanics below CoreML; the bandwidth characterization tools are a proxy for ANE throughput that could inform t3rm1nu55-monitorplus's ANE activity inference.

### Finding 4: ClF3 blog — M3/M4 PMU ESR register format differs from M1/M2
- **Source:** ClF3 blog (new source, not previously tracked)
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** 2026 (exact date not confirmed; documents M3/M4 hardware)
- **Summary:** Documents that on M3 and M4, the PMU event select registers (`SYS_APL_PMESR0_EL1`, `SYS_APL_PMESR1_EL1`) are 64-bit with 16-bit event fields, versus 8-bit event fields on M1/M2. Also notes that `SYS_APL_PMCR0_EL1` is periodically overwritten by a kernel process with a period of ~100 µs, meaning any EL1 modification to PMCR0 will not persist.
- **Why it matters:** Directly actionable for the kperf sidecar: M3/M4 counter configuration requires a different PMESR bitmask than M1/M2, and the PMCR0 overwrite cadence is a hard constraint on any EL1-based counter access.

### Finding 5: macmon — expose active residency ratios via IOReport
- **Source:** vladkens/macmon GitHub (commit 3010f1fb, June 9, 2026)
- **URL:** https://github.com/vladkens/macmon/commit/3010f1fb7e209c334a1f948ea4386d92e1d761d2
- **Date:** June 9, 2026
- **Summary:** macmon (upstream Rust IOReport reference for t3rm1nu55-monitorplus) added per-cluster/core active residency ratio exposure, along with fan speed metrics and a public library API cleanup (v0.8+). Active residency ratios reflect the fraction of time each CPU cluster spends active at each DVFS state, derived from IOReport counters.
- **Why it matters:** Confirms a new IOReport channel is publicly accessible; our own IOReport channel enumeration should be cross-checked against macmon's implementation to ensure we're not missing this metric.

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
