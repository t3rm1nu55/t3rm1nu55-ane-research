# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-04 — sweep (5 findings)

### Finding 1: arXiv 2606.22283 — "Apple Neural Engine: Architecture, Programming, and Performance"
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** 2026-06-21
- **Summary:** A comprehensive reverse-engineered account of the Apple Neural Engine documenting the full dispatch route below CoreML, compiler pipeline, on-disk program format, weight-compression scheme, kernel driver, firmware, and command protocol. Published by Spencer H. Bryngelson et al. (Georgia Tech / Woodruff School). Hacker News discussion at https://news.ycombinator.com/item?id=48702825.
- **Why it matters:** The kernel driver and command protocol documentation is the most complete public treatment of ANE internals ever published; it is a prerequisite reference for implementing any ANE utilization counter in t3rm1nu55-monitorplus.

### Finding 2: arXiv 2606.17090 — ANEForge: Python for direct computation on the Apple Neural Engine
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** 2026-06-12
- **Summary:** ANEForge is an open Python library (Georgia Tech) that compiles a lazy tensor graph into a single ANE program and dispatches it directly, bypassing CoreML. Achieves a ~70 µs per-program dispatch floor; ResNet-18 forward pass in 0.33 ms. Supports full training (forward + backward + Adam optimizer) on the ANE.
- **Why it matters:** Establishes a new open direct-dispatch path to the ANE with a known dispatch overhead floor; closely related to arXiv 2606.22283 and likely uses the same kernel driver pathway documented there.

### Finding 3: arXiv 2603.06728 — Orion: Characterizing and Programming Apple's Neural Engine
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** 2026-03-06
- **Summary:** First open end-to-end runtime for ANE inference and training bypassing CoreML via `_ANEClient` and `_ANECompiler` private APIs. Catalogs 14 previously undocumented ANE MIL IR program constraints and references `_ANESharedEvents` as a sync/fence primitive. Achieves 94% ANE utilization on deep (16–64 op) graphs, measured as achieved TFLOPS / theoretical peak.
- **Why it matters:** `_ANESharedEvents` is a newly documented private API surface worth examining for event-driven utilization hooks; the 14 constraint additions are the most current public account of what the ANE compiler accepts.

### Finding 4: arXiv 2606.25426 — "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX"
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** 2026-06 (exact day pending full paper access)
- **Summary:** Hand-written GEMM kernel that outperforms Apple Accelerate on M1 AMX by exploiting idle AMX blocks. Introduces a "direct per-core occupancy probe" that demonstrates Accelerate holds the E-cluster AMX block at zero utilization while the proposed kernel drives it. M1 AMX inner loop is characterised as load-issue bound.
- **Why it matters:** The per-core AMX occupancy probe is the first published direct AMX block utilization measurement seen in this survey; if the probe uses a kperf event or register read, this is the highest-priority lead for AMX counter support in t3rm1nu55-monitorplus.

### Finding 5: macmon — new active residency ratios metric (IOReport DVFS channel)
- **Source:** vladkens/macmon
- **URL:** https://github.com/vladkens/macmon/commit/3010f1fb
- **Date:** 2026-06-09
- **Summary:** macmon now exposes CPU P-state active residency ratios sourced from IOReport's DVFS channel (commit `3010f1fb`, refs issue #61). Released as part of a larger v0.8 feature batch alongside fan names and per-core CPU view.
- **Why it matters:** Confirms the DVFS residency channel is being actively consumed upstream; warrants checking whether t3rm1nu55-monitorplus should surface the same metric alongside its kperf frequency data.

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
