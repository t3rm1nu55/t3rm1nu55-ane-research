# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-26 — sweep (6 findings)

### Finding 1: Bryngelson — "Apple Neural Engine: Architecture, Programming, and Performance" (arXiv 2606.22283)
- **Source:** arXiv / sbryngelson/ane-guide (GitHub)
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 2026
- **Summary:** Comprehensive reference paper documenting the ANE's datapath, roofline, dispatch route below CoreML, compiler pipeline, program format (`.mil`/`.mlpackage` internals), and kernel driver. Derived from measurement and decompilation; the associated GitHub repo (`sbryngelson/ane-guide`) is actively maintained. This is the most thorough public treatment of ANE internals to date.
- **Why it matters:** Essential reference for any future ANE counter reverse engineering; the kernel driver section may clarify whether performance counter registers are accessible from host.

### Finding 2: Orion — ANE characterization for LLM inference (arXiv 2603.06728)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 2026 (missed in initial April 7 sweep)
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference." Finds that deep operation graphs (16–64 ops) achieve 94% ANE utilization. Extends the public catalog of ANE dispatch constraints to 20 rules (14 newly documented), covering MIL IR, memory, and I/O restrictions.
- **Why it matters:** Establishes via black-box benchmarking what "full ANE utilization" looks like; useful calibration for any power-based ANE utilization proxy in monitorplus.

### Finding 3: AMX dual-block structure revealed (arXiv 2606.25426)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" (Deyvik Bhan). Finds that the M1 has two independent on-chip AMX blocks that Accelerate underutilizes; the inner loop is load-issue bound, capping single-thread throughput at 610–680 GFLOPS. A direct-AMX kernel using fine multi-thread panels and weight pre-packing beats Accelerate/BNNS by 1.17×.
- **Why it matters:** First public documentation of M1 having dual AMX blocks; any future AMX counter work needs to account for two independent coprocessor instances per P-core cluster.

### Finding 4: kperf/kpc counter methodology demonstrated on M4 Pro (arXiv 2606.27098)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.27098
- **Date:** June 2026
- **Summary:** "Residual GPU Cache State on Apple M4 Pro" (Alpay & Başaran). Uses the kperf/kpc private interface as root to program M4 Pro PMU counters, recovering L1D refill granularity (64 B), L1D capacity (128 KiB), and a full empirical memory-hierarchy curve via counter-driven measurement rather than microbenchmarks alone.
- **Why it matters:** Directly demonstrates the kperf/kpc root-access programming pattern on M4; the paper's methodology (counter selection, sampling loop, result interpretation) is a reference implementation for our own privileged sidecar counter work.

### Finding 5: ri_neural_footprint — public API for per-process ANE memory (kennss/SiliconScope)
- **Source:** GitHub — kennss/SiliconScope
- **URL:** https://github.com/kennss/SiliconScope
- **Date:** 2026 (active; commit count 240+)
- **Summary:** SiliconScope's Process Inspector exposes per-process Neural-Engine memory via `ri_neural_footprint` from the public `proc_pid_rusage()` syscall — no root, no private APIs. The ANE "utilization" gauge is a power-normalized estimate from IOReport Energy Model (same as our approach), but the per-process ANE memory field is a genuinely new public-API data point not currently in monitorplus.
- **Why it matters:** `ri_neural_footprint` is a public SDK call we can add to the main project today without privileged access; it's a step toward per-process ANE telemetry even before hardware counter exposure is solved.

### Finding 6: maderix Part 3 — full ANE training (forward + backward pass) via private APIs
- **Source:** maderix Substack
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** 2026 (after Part 2, exact date unconfirmed)
- **Summary:** Continues the M4 ANE reverse-engineering series with a full training implementation: forward pass, backward pass, gradient computation, and Adam optimizer for a 109M-parameter model, later scaled to Qwen3-0.6B. Identifies that ANE's native SDPA ignores causal masks, requiring dispatch decomposition across ANE and CPU.
- **Why it matters:** The depth of ANE dispatch understanding in this work (including gradient flow through `_ANEClient`) may surface new private symbols or scheduling constraints relevant to ANE counter access attempts.

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
