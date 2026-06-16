# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-16 — sweep (3 findings)

### Finding 1: Orion — first academic system for direct ANE programming (arXiv 2603.06728)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026
- **Summary:** Ramchand Kumaresan's "Orion" paper presents the first open end-to-end system for direct ANE execution, bypassing CoreML entirely via Apple's private `_ANEClient` and `_ANECompiler` APIs. It catalogs 20 restrictions on MIL IR programs covering memory layout, compilation limits, and numerical behavior — the most systematic public characterization of the ANE's private API surface to date. The system demonstrates LLM training and inference on the ANE with checkpoint resume, and cites maderix's reverse-engineering work as its empirical foundation.
- **Why it matters:** The 20-restriction constraint catalog extends the known `_ANEClient` API surface; any future ANE activity detection in t3rm1nu55-monitorplus (e.g., via dispatch pattern or compilation event hooks) must account for these constraints.

### Finding 2: maderix Part 3 — ANE training with backward pass and Adam optimizer
- **Source:** maderix Substack
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** March 2026
- **Summary:** Third installment of the maderix M4 ANE series; demonstrates full transformer training on the ANE with forward pass, backward pass, gradient computation, and Adam optimizer updates, scaling to Qwen3-0.6B (596M parameters). This confirms the ANE's private API supports stateful gradient ops, not just inference dispatch — a non-obvious capability that significantly expands the known API surface.
- **Why it matters:** Training requires new `_ANEClient` call paths not seen in the first two parts; any IOReport energy-based ANE detection heuristic should now account for sustained dual-direction dispatch patterns distinct from inference-only workloads.

### Finding 3: ClF3 blog — M3/M4 PMU ESR register encoding differs from M1/M2
- **Source:** ClF3's blog
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** 2026 (exact date unknown)
- **Summary:** Empirical analysis of kperf/kpc on M3 and M4 reveals a breaking hardware difference: the ESR configuration registers are 64-bit on M3/M4 (vs 32-bit on M1/M2), and each event slot occupies 16 bits (vs 8 bits on M1/M2). This means the event-packing code in any kperf FFI must branch on chip generation or it will silently misconfigure counters on M3 and M4. The bugsiki post (already tracked) did not surface this encoding delta.
- **Why it matters:** The kperf sidecar in t3rm1nu55-monitorplus must handle two distinct ESR layouts; running M1/M2 packing logic on an M3/M4 host will produce wrong counter reads with no runtime error.

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
