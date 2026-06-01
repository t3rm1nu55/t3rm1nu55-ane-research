# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-01 — sweep (3 findings)

### Finding 1: Asahi m1n1 adds PMGR support for M5 and A18 Pro Mac
- **Source:** AsahiLinux/m1n1
- **URL:** https://github.com/AsahiLinux/m1n1/commit/fcaf4765c443d4e7432e470e40c25a1801217f40
- **Date:** 2026-05-05 (merged 2026-05-15)
- **Summary:** The Asahi team landed "pmgr: support M4 Pro/Max / A18 Pro / M5" — power-manager register-map support for Apple's newest chips in m1n1. A companion commit ("Initial support for T8140", likely M5 Max) landed 2026-05-03. The work includes updated ADT power-state-group parsing and a corrected MMIO layout for secondary watchdogs, indicating active hardware bring-up on these SoCs.
- **Why it matters:** IOReport's "Energy Model" channel on macOS reads from the same PMGR infrastructure; new chip generations can shift register offsets and channel names, silently breaking ANE energy telemetry in t3rm1nu55-monitorplus.

### Finding 2: maderix publishes Part 3 + open-sources `maderix/ANE` training framework (missed in seed)
- **Source:** maderix Substack / github.com/maderix/ANE
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b · https://github.com/maderix/ANE
- **Date:** March 2026
- **Summary:** Part 3 of the M4 ANE series covers full transformer training on the ANE. The accompanying open-source repo uses three private APIs — `_ANEClient`, `_ANECompiler`, and `_ANEInMemoryModelDescriptor` — to compile MIL directly to ANE programs without writing disk-based `.mlmodelc` files. Benchmarks document INT8 W8A8 throughput at 35.1 TOPS (vs 18.6 TOPS FP16), a ratio not previously public.
- **Why it matters:** `_ANEInMemoryModelDescriptor` is a newly documented symbol that enables compile-time dispatch control; the repo's benchmark harness could serve as a basis for a dispatch-timing proxy for ANE utilization inference.

### Finding 3: Orion paper catalogs ANE compilation constraints (arXiv 2603.06728, missed in seed)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** 2026-03-06
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" (Kumaresan, 2026) is the first academic paper to document 20 MIL IR constraints governing what can execute on the ANE, and builds an end-to-end training runtime that bypasses CoreML via `_ANEClient`/`_ANECompiler`. The "94% ANE utilization" figure is a throughput proxy (measured FLOPS / theoretical peak), not a hardware counter — confirming that no counter path has been discovered.
- **Why it matters:** Provides a canonical, citable reference for ANE compilation constraints; the negative result on hardware counters sets the definitive baseline for the field as of early 2026.

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
