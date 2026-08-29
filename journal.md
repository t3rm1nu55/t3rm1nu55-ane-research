# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-29 — sweep (4 findings)

### Finding 1: "Apple Neural Engine: Architecture, Programming, and Performance" (arXiv 2606.22283)
- **Source:** arXiv / sbryngelson
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 21, 2026
- **Summary:** Spencer Bryngelson's comprehensive reverse-engineered account of the ANE, based on direct measurement on Apple Silicon and static analysis of the private runtime, compiler, kernel driver, and firmware. Documents the datapath and throughput/energy roofline, the dispatch route below Core ML, the compiler and on-disk program format, the weight-compression scheme, and the full kernel-driver/firmware/command protocol. Companion guide at `sbryngelson/ane-guide` and toolkit at `sbryngelson/ANEForge`.
- **Why it matters:** First public documentation of the ANE kernel-driver command protocol — the layer where hardware performance counters, if they exist, would be accessible. Directly expands the search space for ANE counter discovery.

### Finding 2: "ANEForge: Python for direct computation on the Apple Neural Engine" (arXiv 2606.17090)
- **Source:** arXiv / sbryngelson
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 2026
- **Summary:** Python package (available on PyPI) that compiles lazy tensor graphs from 58 fused operators into a single ANE program, dispatched via the same daemon/kernel-driver stack as Apple's internal framework. Exposes 18+ native ANE layers not reachable through CoreML, and runs training (forward + backward + Adam) entirely on-chip.
- **Why it matters:** Programmable surface above the raw command protocol documented in 2606.22283; if any counter readback mechanism exists in the IOKit driver, this stack is the right place to probe it.

### Finding 3: "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" (arXiv 2603.06728)
- **Source:** arXiv / mechramc
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 2026
- **Summary:** First open end-to-end ANE system for LLM training and inference bypassing CoreML via `_ANEClient`/`_ANECompiler` private APIs. Compiler lowers a graph IR through 5 optimization passes to ANE-native MIL; runtime uses IOSurface-backed zero-copy tensor I/O. Achieves 170+ tokens/s GPT-2 inference and 8.5× speedup over naive per-step recompilation via weight-patching. GitHub: `mechramc/Orion`.
- **Why it matters:** Open implementation of the `_ANEClient` dispatch path confirms the private API is stable enough for production use; establishes a reference for how to instrument ANE execution from the host side.

### Finding 4: "Utilizing PMU Event Counters on Apple M3 and M4" (ClF3's blog)
- **Source:** blog.clf3.org
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** 2026 (exact date unavailable — site blocked from fetch)
- **Summary:** Blog post documenting differences in PMU register definitions between M1/M2 and M3/M4, including 64-bit counter width and new PMI behaviour on M3/M4. Notes that chip-specific kperf event catalogs live at `/usr/share/kpep/` as plist files (e.g. `as4.plist` for M4), distinct from the M1/M2 catalogs. The full event list for M3/M4 is documented in the post.
- **Why it matters:** M3/M4 kperf event catalog differences are directly relevant to our kperf PMU counter integration — our sidecar must load the correct chip-specific plist and account for the wider counter registers on M3+.

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
