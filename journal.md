# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-29 — sweep (4 findings)

### Finding 1: Comprehensive ANE architecture paper — datapath, firmware, command protocol
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 21, 2026
- **Summary:** Spencer Bryngelson's reverse-engineered account of the full ANE software stack: datapath and roofline, dispatch route below CoreML via `_ANEClient`/`_ANECompiler`, the compiler and on-disk MIL format, weight-compression scheme, kernel driver, firmware, and command protocol — all derived from direct measurement and static analysis of the private runtime. Received HN attention (item 48702825).
- **Why it matters:** Most complete public documentation of ANE internals to date; the kernel driver and firmware analysis is the first place to look for whether hardware performance counter registers exist at the driver level.

### Finding 2: Orion — first open end-to-end system for direct ANE compute dispatch
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026
- **Summary:** Ramchand Kumaresan's "Orion" bypasses CoreML entirely via `_ANEClient`/`_ANECompiler`, adds a compiler pipeline, and achieves stable multi-step LLM training with checkpoint resume. Catalogs 20 constraints on MIL IR programs and reports 94% ANE utilization on deep operation graphs (16–64 ops), 170+ tok/s on GPT-2 124M.
- **Why it matters:** The 20-constraint MIL IR catalog is the most complete public specification of ANE program requirements; confirms private API stability for arbitrary compute dispatch well beyond maderix's initial proof-of-concept.

### Finding 3: maderix Part 3 — full transformer training on ANE + open-source code released
- **Source:** maderix Substack / GitHub
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b / https://github.com/maderix/ANE
- **Date:** March 2026
- **Summary:** Part 3 of the tracked maderix series demonstrates complete transformer training (forward pass, backward pass via `dx` on ANE, `dW` on CPU via cblas, Adam optimizer) on 109M-parameter models up to Qwen3-0.6B (596M params). All code is open-sourced at `maderix/ANE`, tested on M4 Mac Mini under macOS 15.x.
- **Why it matters:** First publicly available code for arbitrary ANE graph dispatch including backprop; directly complements Orion and confirms `_ANEClient`/`_ANECompiler` are stable enough for production-style use.

### Finding 4: ClF3 blog — M3/M4 PMU ESR registers are 64-bit with 16-bit event fields
- **Source:** blog.clf3.org
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** Unknown (late 2025 / early 2026, after bugsiki January 2026 post)
- **Summary:** Documents a breaking hardware register change between generations: M3 and M4 PMU ESR registers are 64-bit with 16-bit fields per event slot, versus 8-bit fields on M1/M2. Enabling PMC2–PMC9 requires setting bits in `SYS_APL_PMCR0_EL1` and `SYS_APL_PMCR1_EL1`; the bit assignment in those control registers also differs from earlier chips.
- **Why it matters:** Any kperf integration that hard-codes M1/M2 8-bit event field widths will silently misread counter configurations on M3/M4 — this is an actionable correctness bug to audit in the kperf sidecar.

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
