# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-30 — sweep (4 findings)

### Finding 1: Comprehensive ANE Architecture, Programming, and Performance paper (M1–M5)
- **Source:** arXiv / Spencer H. Bryngelson (Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.22283 — companion guide: https://ane-guide.readthedocs.io
- **Date:** June 21, 2026
- **Summary:** Reverse-engineered account of the ANE based on direct hardware measurement and static analysis of the private runtime, compiler, kernel driver, and firmware. Covers A11 through M5 families with direct measurements on M1 and M5. Documents the full dispatch path below CoreML: the datapath roofline, compiler and on-disk program format, weight-compression scheme, kernel driver command protocol, and firmware ABI. An accompanying reference manual is published at ane-guide.readthedocs.io.
- **Why it matters:** Most thorough public documentation of ANE internals to date; the kernel driver and firmware command protocol sections are directly relevant to any future attempt to surface ANE utilization metrics in t3rm1nu55-monitorplus.

### Finding 2: ANEForge — Python framework for direct ANE computation without CoreML
- **Source:** arXiv / Spencer H. Bryngelson (Georgia Tech); PyPI: aneforge; GitHub: sbryngelson/ANEForge
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 12, 2026
- **Summary:** Open Python package that compiles a lazy tensor graph (58 fused operators + 19 native bridge operators) into an ANE program and dispatches it through the same ANE daemon and kernel-driver stack Apple uses internally, bypassing CoreML entirely. ANEForge is the first public tool that places arbitrary compute on the ANE without CoreML as intermediary.
- **Why it matters:** Actionable — the dispatch path ANEForge exposes is exactly where ANE utilization signals would live. Studying ANEForge's kernel-driver interaction could provide the approach for reading ANE counters or power state in t3rm1nu55-monitorplus.

### Finding 3: M5 Neural Accelerators exposed via Metal 4 tensor API (BaseRT paper)
- **Source:** arXiv / Waschkowski, Rathnayaka, Wesemann (Base Compute)
- **URL:** https://arxiv.org/abs/2607.19438
- **Date:** July 21, 2026
- **Summary:** The M5 GPU architecture places a dedicated Neural Accelerator on every GPU core, exposed through the public Metal 4 tensor API. The BaseRT LLM inference runtime routes matrix multiplications through these units and achieves 6.4× higher prefill throughput than llama.cpp. Unlike the ANE (private API only), M5 neural accelerators are reachable via a documented Metal API.
- **Why it matters:** M5 may represent a new paradigm where neural compute is measurable via Metal performance counters rather than reverse-engineered ANE private APIs — warrants investigation for M5 support in t3rm1nu55-monitorplus.

### Finding 4: AMX inner-loop characterization on M1 — load-issue bound
- **Source:** arXiv / Deyvik Bhan
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Characterizes single-precision GEMM on M1 AMX and finds the inner loop is load-issue bound, with single-thread throughput at 610–680 GFLOPS. Demonstrates a hand-written kernel that exceeds Accelerate's BNNS Graph path by 1.17× by exploiting M1's second on-chip AMX block via fine multi-thread panels. Builds on the MIT AMX thesis (Sep 2025).
- **Why it matters:** The load-issue bound finding identifies which PMU event class (load-related) would be most diagnostic for AMX utilization — directly relevant to the AMX event discovery workstream in t3rm1nu55-monitorplus.

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
