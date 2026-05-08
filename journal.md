# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-08 — sweep (3 findings)

### Finding 1: Orion — systematic catalog of 20 ANE compiler constraints (arXiv 2603.06728)
- **Source:** arXiv (academic preprint)
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026 (missed in initial seed)
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" is the most comprehensive academic characterization of ANE behavior to date, cataloging 20 constraints on MIL IR programs (memory layout rules, numerical restrictions, graph depth limits, and the ~119-compilation-per-process hard cap). It builds directly on the maderix `_ANEClient` reverse engineering and extends it into a complete LLM training and inference runtime that bypasses CoreML entirely.
- **Why it matters:** The 119-compile-per-process cap is a hard architectural constraint for any monitoring tool that probes ANE activity by dispatching workloads — relevant to future t3rm1nu55-monitorplus ANE probing strategies.

### Finding 2: maderix ANE GitHub repo + Part 3 — training on ANE via three private symbols
- **Source:** maderix Substack / GitHub (maderix/ANE)
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b / https://github.com/maderix/ANE
- **Date:** March 7, 2026 (missed in initial seed)
- **Summary:** Part 3 of the maderix series demonstrates a full gradient-descent training loop (forward pass, backward pass, Adam optimizer) for a 109M-parameter transformer running entirely on the M4 ANE. The accompanying GitHub repo shows the complete working implementation using three private symbols: `_ANEClient`, `_ANECompiler`, and `_ANEInMemoryModelDescriptor` — the third symbol not previously documented in the seed.
- **Why it matters:** Adds `_ANEInMemoryModelDescriptor` to the known private API surface; confirms that M4 ANE compute is programmable without CoreML, which is foundational context for any future direct-dispatch ANE utilization probe.

### Finding 3: macpow (k06a/macpow) — new IOReport reference covering M1–M5+ with channel dump mode
- **Source:** GitHub (k06a/macpow)
- **URL:** https://github.com/k06a/macpow
- **Date:** v0.1.17 released April 9, 2026 (post last-checked date; previously untracked)
- **Summary:** macpow is an untracked IOReport-based power TUI supporting M1 through M5+ (including multi-die Ultra variants) without requiring root. It handles single-die vs. multi-die IOReport channel name differences (e.g., `ANE` vs. `ANE0_0` on Ultra chips) and exposes a `--dump` flag that prints all raw IOReport channel names from the live hardware — making it a practical tool for enumerating undocumented channels on new chip generations as they ship.
- **Why it matters:** The `--dump` mode is a direct mechanism for discovering IOReport channel names on M5 hardware that t3rm1nu55-monitorplus does not yet handle; the multi-die channel naming logic is a reference for Ultra chip support.

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
