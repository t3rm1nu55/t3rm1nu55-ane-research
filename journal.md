# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-24 — sweep (4 findings)

### Finding 1: "Apple Neural Engine: Architecture, Programming, and Performance" (arXiv:2606.22283)
- **Source:** arXiv / GitHub (sbryngelson/ane-guide)
- **URL:** https://arxiv.org/abs/2606.22283 · https://github.com/sbryngelson/ane-guide
- **Date:** June 21, 2026
- **Summary:** Spencer Bryngelson (Georgia Tech) published the most complete public reverse-engineering of the ANE to date, derived from direct measurement and static decompilation of the private runtime, compiler, kernel driver, and firmware. The reference documents the full dispatch route below Core ML, register maps, the command protocol between the CPU and ANE firmware, per-chip target tables, and a weight-compression scheme. Companion web edition at ane-guide.readthedocs.io.
- **Why it matters:** Contains the register maps and below-CoreML dispatch path that are the prerequisite for any ANE utilization instrumentation; directly answers "what would we instrument?" for t3rm1nu55-monitorplus v2.

### Finding 2: ANEForge — Python for direct ANE computation without CoreML (arXiv:2606.17090)
- **Source:** arXiv / GitHub (sbryngelson/ANEForge)
- **URL:** https://arxiv.org/abs/2606.17090 · https://github.com/sbryngelson/ANEForge
- **Date:** June 12, 2026
- **Summary:** Same author as Finding 1. ANEForge is a Python package that compiles a lazy tensor graph (58 fused + 19 bridge operators) to native ANE programs and dispatches them through the same daemon/kernel-driver stack Apple uses internally, bypassing Core ML entirely. Demonstrates training (forward + backward + optimizer) on-chip. Dispatch floor is ~70 µs; a ResNet-18 forward pass runs end-to-end in 0.33 ms.
- **Why it matters:** The dispatch path is now public and library-quality — the same path is the one that would need to be instrumented for real-time utilization tracking. ANEForge is the Rust FFI reference model for the main project's future ANE sidecar.

### Finding 3: M4 kperf database exposes SME engine hardware counters (jiegec/apple-pmu)
- **Source:** GitHub (jiegec/apple-pmu) — July 3, 2026 "Add analysis" commit; this repo is untracked
- **URL:** https://github.com/jiegec/apple-pmu · https://github.com/jiegec/apple-pmu/blob/master/as4.md
- **Date:** July 3, 2026 (latest commit); M4 as4.md content extracted from macOS 26.x kpep database
- **Summary:** The jiegec/apple-pmu repo dumps the `/usr/share/kpep` PMU event databases for every Apple Silicon generation. The M4 database (as4.md) exposes four new `INST_SME_ENGINE_*` kperf events: `ALU`, `LD`, `ST`, and `SCALARFP` — retiring SME (ARM Scalable Matrix Extension) instructions. These are the first documented hardware performance counter events for matrix-accelerator-class operations on Apple Silicon, accessible via the standard kperf/kpc interface without additional reverse engineering. M5 (as5.md) extends this with load data source tracking and PL2 cache events.
- **Why it matters:** On M4+, matrix acceleration via SME is now directly countable with existing kperf infrastructure — no new reverse engineering needed. This is a partial answer to the "AMX utilization" open problem for M4 and later chips; the main project can expose `INST_SME_ENGINE_ALU` (and siblings) via its existing kperf sidecar.

### Finding 4: "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" (arXiv:2606.25426)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Demonstrates a direct-AMX fp32 GEMM kernel that exceeds all Accelerate paths by 1.09–2.0× across 12 LLM prefill shapes on M1, by exploiting both M1 AMX blocks and pre-packed weight layout that Accelerate underuses. Results are bit-identical to Accelerate. This is the most detailed public characterization of AMX block utilization yet, though it does not expose new hardware counters.
- **Why it matters:** Confirms M1 has two AMX blocks (not one) and quantifies their utilization model; useful context for interpreting any future AMX counter data and for calibrating power-based AMX utilization inference.

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
