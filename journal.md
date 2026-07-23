# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-23 — sweep (6 findings)

### Finding 1: ANEForge — Python direct ANE dispatch without CoreML

- **Source:** arXiv / GitHub (sbryngelson/ANEForge)
- **URL:** https://arxiv.org/abs/2606.17090 · https://github.com/sbryngelson/ANEForge
- **Date:** 2026-06-12
- **Summary:** Spencer Bryngelson (Georgia Tech) released ANEForge, a Python package that compiles a lazy tensor graph of 58 fused operators into a native ANE program and dispatches it through the same `_ANEClient` daemon/driver stack as Apple's internal runtime — entirely bypassing CoreML. The paper measures ANE dispatch latency at a 70 µs per-program floor, with a fused program completing in ~90 µs. Training on the ANE (forward pass, backward pass, optimizer) is demonstrated for the first time.
- **Why it matters:** Dispatch-latency polling is now a demonstrated utilization proxy; the 70 µs floor gives a reference point for detecting ANE saturation.

### Finding 2: "Apple Neural Engine: Architecture, Programming, and Performance" — comprehensive reverse-engineering reference

- **Source:** arXiv / GitHub (sbryngelson/ane-guide)
- **URL:** https://arxiv.org/abs/2606.22283 · https://github.com/sbryngelson/ane-guide · https://ane-guide.readthedocs.io
- **Date:** 2026-06-21
- **Summary:** Companion to ANEForge, this paper documents the ANE's datapath and roofline, the full dispatch route from userspace to firmware (bypassing CoreML), the on-disk E5 binary program format, the weight-compression scheme, the kernel driver, and the command protocol. It is the most complete public reverse-engineering reference for any Apple ANE generation.
- **Why it matters:** Provides the architecture grounding needed to reason about what registers or counters a hypothetical ANE PMU driver would need to read; also the canonical citation for any future monitorplus ANE work.

### Finding 3: "Above the Inner Loop" — M1 AMX has two on-chip blocks

- **Source:** arXiv (2606.25426)
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** ~2026-06-25
- **Summary:** Deyvik Bhan (Georgia Tech) shows that the M1 P-cluster contains two AMX blocks and that Apple's Accelerate library leaves the second block idle for certain matrix shapes. A direct-AMX GEMM kernel that fills both blocks achieves 1.17–1.58× over Accelerate fp32 paths across LLM prefill shapes (M1–M3).
- **Why it matters:** Any future AMX utilization metric must account for two independent AMX blocks per P-cluster, not one; single-block saturation ≠ full AMX utilization.

### Finding 4: macmon exposes DVFS active residency ratios (IOReport)

- **Source:** GitHub (vladkens/macmon)
- **URL:** https://github.com/vladkens/macmon/commit/3010f1fb7e209c334a1f948ea4386d92e1d761d2
- **Date:** 2026-06-09
- **Summary:** macmon v0.8 added `expose active residency ratios` (issue #61), pulling per-cluster DVFS state-time fractions from IOReport's `CPU Core Performance States` channel. This is the first clean Rust implementation of this channel in our tracked tools.
- **Why it matters:** t3rm1nu55-monitorplus does not yet expose active residency ratios; this upstream implementation is the reference to port from.

### Finding 5: ClF3 blog — M3/M4 PMU ESR fields are 16-bit, not 8-bit

- **Source:** ClF3's blog
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** 2026 (exact date not retrieved)
- **Summary:** A new technical blog post documents that M3 and M4 change the PMU Event Select Register (ESR) layout: each event field is 16-bit (vs 8-bit on M1/M2), fitting 4 events per 64-bit register. The M4's kpep database file is `as4.plist`. M3/M4 retain 2 fixed + 8 configurable counters (10 total), same as M1/M2.
- **Why it matters:** The kperf FFI in t3rm1nu55-monitorplus must detect chip generation and use the correct ESR bit-field width, or counter programming will silently corrupt event selection on M3/M4.

### Finding 6 (gap-fill): Orion — first open LLM training runtime on ANE without CoreML

- **Source:** arXiv / GitHub (mechramc/Orion)
- **URL:** https://arxiv.org/abs/2603.06728 · https://github.com/mechramc/Orion
- **Date:** 2026-03-06 (pre-dates tracking; not captured in seed)
- **Summary:** Ramchand Kumaresan published Orion, the first open end-to-end system for training and inference of LLMs directly on the ANE, using `_ANEClient` and `_ANECompiler` private APIs. It predates ANEForge (Finding 1) and is cited by it. The HN thread (item 47257931) had significant traction.
- **Why it matters:** Establishes that direct ANE training is reproducible by independent groups (confirmed by ANEForge); the `_ANEClient`/`_ANECompiler` API path is now the consensus mechanism.

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
