# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-07 — sweep (6 findings)

### Finding 1: "Apple Neural Engine: Architecture, Programming, and Performance" (arXiv 2606.22283)
- **Source:** arXiv / ane-guide.readthedocs.io
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** 2026-06-21
- **Summary:** Spencer Bryngelson published a comprehensive reverse-engineered reference manual for the ANE covering the full stack: fp16 datapath, roofline, dispatch route below CoreML, compiler/program format, weight-compression scheme, kernel driver, firmware, and command protocol. A web edition lives at ane-guide.readthedocs.io. Based on direct measurement on Apple silicon and static analysis of the private runtime and kernel extension.
- **Why it matters:** Kernel driver + firmware + command protocol documentation is the missing link for inferring ANE utilization — this is the most complete public reference to date and should be treated as the new ground truth for any ANE counter work.

### Finding 2: ANEForge — Python direct ANE dispatch (arXiv 2606.17090)
- **Source:** arXiv / PyPI / GitHub (sbryngelson/ANEForge)
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** 2026-06-12
- **Summary:** ANEForge is a Python package (available on PyPI) that compiles a lazy tensor graph of 58 fused operators + 19 native bridge operators into a single ANE program and dispatches it directly, bypassing CoreML entirely. Authored by the same group as the ANE architecture guide.
- **Why it matters:** First installable open-source SDK for direct ANE dispatch; exposes the same private API surface we'd need for a Rust FFI in t3rm1nu55-monitorplus and confirms the viability of below-CoreML ANE access.

### Finding 3: Orion — open LLM training/inference on ANE (arXiv 2603.06728)
- **Source:** arXiv / GitHub (mechramc/Orion)
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** 2026-03-06
- **Summary:** Orion is the first open end-to-end system combining direct ANE execution, a compiler pipeline, and stable multi-step LLM training using Apple's private `_ANEClient` and `_ANECompiler` APIs without CoreML. Achieves 170+ tokens/s for GPT-2 124M on M4 Max and reduces recompilation from 4,200 ms to 494 ms per step (8.5x).
- **Why it matters:** Third independent open implementation proving direct `_ANEClient` access is stable; validates the dispatch path documented in the ANE guide above.

### Finding 4: maderix/ANE — ANE training with utilization metrics
- **Source:** GitHub (maderix/ANE)
- **URL:** https://github.com/maderix/ANE
- **Date:** 2026 (Q1–Q2)
- **Summary:** Open repository implementing transformer forward/backward passes directly on ANE via `_ANEClient`/`_ANECompiler`/`_ANEInMemoryModelDescriptor`. Reports concrete ANE utilization figures derived from FLOPs accounting: ~5–9% utilization at 18.6 TFLOPS FP16 peak on M4. Also documents that INT8 W8A8 achieves 1.88× over FP16.
- **Why it matters:** First public report of a quantified ANE utilization percentage, derived by comparing measured FLOPs against hardware peak — establishes that FLOPs-based utilization estimation is viable even without hardware counters.

### Finding 5: "Above the Inner Loop" — AMX GEMM characterization (arXiv 2606.25426)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** 2026-06-24
- **Summary:** Characterizes M1 AMX inner loop behavior for LLM prefill GEMMs. Finds the M1 has at least two on-chip AMX blocks; the inner loop is load-issue bound (610–680 GFLOPS with any operand load interleaved, vs. full rate without). A custom direct-AMX kernel beats Accelerate's fastest FP32 path by 1.17× using fine multi-thread panels and weight pre-packing.
- **Why it matters:** Confirms M1 AMX block count and bound type through microbenchmarking; no hardware counter exposure found, but load-issue binding implies future AMX utilization inference via L1D miss counters is worth investigating.

### Finding 6: mperf — new perf-stat-like CLI for Apple Silicon kperf
- **Source:** lambdafoo.com / Perpetually Curious Blog
- **URL:** https://lambdafoo.com/posts/2026-03-25-mperf-hardware-counters-macos.html
- **Date:** 2026-03-25
- **Summary:** `mperf` is a new `perf stat`-like CLI for Apple Silicon built on kperf/kperfdata private frameworks. Supports up to 10 simultaneous counters with JSON output, suitable for scripting. Exposes standard events (cycles, instructions, L1D TLB misses) without sudo.
- **Why it matters:** Another reference implementation for kperf user-space access, newer than the ibireme gist; the JSON output mode is directly relevant to how our privileged sidecar should expose raw counter data.

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
