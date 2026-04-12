# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-04-12 — sweep (3 findings)

*Note: All three findings predate the April 7 seed date (they are from Feb–Mar 2026) but were absent from the seed journal. This is the first automated sweep; they are logged here to bring the journal current.*

### Finding 1: maderix Part 3 + open-source ANE training/benchmark repo
- **Source:** maderix Substack + GitHub
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b / https://github.com/maderix/ANE
- **Date:** March 7, 2026 (Part 3 post); repo created February 28, 2026
- **Summary:** Part 3 completes the maderix M4 ANE series with a full transformer training walkthrough — forward pass, backward pass, Adam optimizer on 109 M parameters — entirely on ANE via `_ANEClient`/`_ANECompiler` private APIs. The accompanying `maderix/ANE` repo ships benchmark utilities including ANE SRAM bandwidth probing, INT8 vs FP16 throughput comparisons, and a live training dashboard; it has also been validated on Qwen3-0.6B (596 M params).
- **Why it matters:** The SRAM bandwidth probing utility is the first public tool to empirically measure ANE memory throughput — a proxy for utilization when hardware counters are absent. The repo's benchmark infrastructure should be audited as a reference for t3rm1nu55-monitorplus's future ANE throughput-inference path.

### Finding 2: Orion — first open end-to-end ANE LLM system + 20-restriction catalog
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026
- **Summary:** Ramchand Kumaresan's "Orion" paper presents the first open, end-to-end system combining direct ANE execution, a compiler pipeline, and stable multi-step training with checkpoint resume — all bypassing CoreML via `_ANEClient`/`_ANECompiler`. The paper catalogs 20 ANE restrictions on MIL IR, memory layout, compilation limits, and numerical behavior, 14 of which were previously undocumented; among them: the ANE compiler silently fails after ~119 compilations per process, and deep operation graphs (16–64 ops) achieve 94% ANE utilization while shallow ones waste significant pipeline capacity.
- **Why it matters:** The 119-compilation per-process limit is an operational constraint any long-running monitoring tool must handle (e.g. by forking a fresh sidecar). The 20-restriction catalog is now the authoritative public reference for safe ANE program construction.

### Finding 3: clf3 blog — M3/M4 PMU ESR register format differs from M1/M2
- **Source:** clf3's blog
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** Unknown (page returned 403; content surfaced via web search snippet)
- **Summary:** Documents a concrete kperf implementation divergence between chip generations: on M1/M2 the PMU ESR register encodes each event in 8 bits, while on M3/M4 the ESR is 64-bit with 16-bit-per-event encoding. This means event-selector code written for M1 will silently select wrong events on M3/M4 if not patched.
- **Why it matters:** The kperf sidecar in t3rm1nu55-monitorplus must apply generation-specific event encoding; this post is the clearest public documentation of the M3/M4 divergence to date. The referenced bugsiki.dev post (already tracked) should be cross-checked to confirm alignment.

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
