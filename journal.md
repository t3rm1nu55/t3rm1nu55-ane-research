# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-22 — sweep (6 findings)

### Finding 1: "Apple Neural Engine: Architecture, Programming, and Performance" (arXiv:2606.22283)
- **Source:** arXiv / Spencer H. Bryngelson (Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.22283 — companion guide: https://ane-guide.readthedocs.io
- **Date:** June 21, 2026
- **Summary:** Reverse-engineered account of the ANE based on direct hardware measurement and static analysis of the private runtime, compiler, kernel driver, and firmware. Documents the datapath, roofline (throughput + energy bounds), dispatch route below CoreML, compiler and on-disk program format, weight-compression scheme, and full kernel driver/firmware/command-protocol stack. Per-chip target tables are included.
- **Why it matters:** The roofline model gives mathematically grounded bounds on ANE throughput per watt — a proxy utilization signal derivable from IOReport energy deltas without hardware counters.

### Finding 2: Orion — direct ANE execution bypassing CoreML (arXiv:2603.06728)
- **Source:** arXiv / Ramchand Kumaresan; GitHub: https://github.com/mechramc/Orion
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026
- **Summary:** First open end-to-end runtime that calls `_ANEClient` and `_ANECompiler` directly, bypassing CoreML entirely, to enable LLM training and inference on the ANE. Achieves 170+ tokens/s for GPT-2 124M on M4 Max; demonstrates stable 110M-parameter transformer training. Builds on maderix's private-API reverse engineering.
- **Why it matters:** Proof that private ANE APIs are stable enough for production use; the dispatch path Orion documents is exactly where a utilization hook would need to sit.

### Finding 3: ANEForge — Pythonic direct-ANE bindings (arXiv:2606.17090)
- **Source:** arXiv / Spencer H. Bryngelson; GitHub: https://github.com/sbryngelson/ANEForge; PyPI: aneforge
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 12, 2026
- **Summary:** Python package that compiles a lazy tensor graph (58 fused + 19 native bridge operators) into a single ANE program and dispatches it through the ANE daemon and kernel-driver stack directly, bypassing CoreML. Guarantees ANE execution where CoreML may silently fall back to CPU/GPU.
- **Why it matters:** Ready-to-instrument code path directly below the CoreML abstraction layer; if ANE performance counters or utilization registers exist, this is the most accessible place to probe them.

### Finding 4: maderix Part 3 — training transformers on the M4 ANE
- **Source:** maderix Substack (Part 3 in series)
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** 2026 (exact date unconfirmed)
- **Summary:** Completes the maderix M4 ANE series by demonstrating transformer training. Maps 40+ private classes in `AppleNeuralEngine.framework` including `_ANEClient`, `_ANECompiler`, and `_ANEInMemoryModelDescriptor`. This is now the most complete public catalog of ANE private API symbols.
- **Why it matters:** The 40-class symbol inventory is a prerequisite for any IOKit/private-framework utilization hook; this is the reference to diff against when new macOS versions ship.

### Finding 5: AMX load-issue bound microarchitecture characterization (arXiv:2606.25426)
- **Source:** arXiv / Deyvik Bhan
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Shows via microbenchmarks that the M1 AMX inner loop is load-issue bound — when any operand load interleaves with the FMA32 stream, single-thread throughput drops to ~610–680 GFLOPS (under half the load-free rate). Exploits M1's second on-chip AMX block and pre-packing to exceed Accelerate's BNNS Graph path by 1.17×. Expected to hold for M1–M3.
- **Why it matters:** The load-issue bound characterization is the first public AMX microarchitecture property expressible as a PMU counter ratio (load stalls vs. retired FP ops), potentially enabling AMX activity detection via existing kperf events.

### Finding 6: macmon v0.8.2 — IOReport interval fix relevant to upstream
- **Source:** vladkens/macmon GitHub
- **URL:** https://github.com/vladkens/macmon
- **Date:** August 4, 2026 (v0.8.2)
- **Summary:** Fixed a regression causing missing per-core CPU metrics on M3 Ultra. The `get_metrics(duration_ms)` call now correctly spans a complete IOReport sampling interval. Earlier July 2026 commits also added a manually-scheduled metrics sampling mode and powermetrics sampling research notes.
- **Why it matters:** t3rm1nu55-monitorplus vendors the IOReport access pattern from macmon. The interval fix directly affects ANE energy-delta accuracy — incomplete intervals produce artificially low energy readings that corrupt the utilization proxy.

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
