# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-05 — sweep (4 findings)

### Finding 1: ANEForge — direct CoreML-free ANE programming library

- **Source:** arXiv / GitHub (sbryngelson/ANEForge)
- **URL:** https://arxiv.org/abs/2606.17090 · https://github.com/sbryngelson/ANEForge
- **Date:** June 12, 2026
- **Summary:** Python library that compiles a lazy operator graph (58 fused operators + 19 native bridge ops unavailable through CoreML) into a single e5rt program and dispatches it directly to the ANE daemon without CoreML. Verified on M1 through M5 (28 ANE targets). Includes tooling for speed and power measurement across chip generations.
- **Why it matters:** First public tool enabling direct ANE throughput and power measurement without CoreML overhead — could complement or cross-validate our IOReport energy-indirection approach.

### Finding 2: Apple Neural Engine Architecture, Programming, and Performance (ane-guide)

- **Source:** arXiv / ReadTheDocs (sbryngelson/ane-guide)
- **URL:** https://arxiv.org/abs/2606.22283 · https://ane-guide.readthedocs.io
- **Date:** June 2026
- **Summary:** Companion reference paper to ANEForge documenting ANE internals from measurement and decompilation: datapath, memory hierarchy, e5rt compiler and program format, kernel driver, firmware, command protocol, and per-chip target tables for M1–M5. Deepest public reverse-engineering of the ANE stack to date.
- **Why it matters:** Documents the dispatch path below CoreML that our current power-indirection approach doesn't rely on — essential reading before evaluating any ANE counter or utilization API surface.

### Finding 3: M4 kperf exposes INST_SME_ENGINE_* counters — matrix ops now countable on M4+

- **Source:** jiegec/apple-pmu (as4.md)
- **URL:** https://github.com/jiegec/apple-pmu/blob/master/as4.md
- **Date:** Documented in /usr/share/kpep/as4.plist (shipped with macOS)
- **Summary:** The M4 replaced Apple's proprietary AMX with ARM's standard SME (Scalable Matrix Extension) and macOS ships SME-specific PMU events in the as4 kpep database: `INST_SME_ENGINE_ALU` (0x8a3), `INST_SME_ENGINE_LD` (0x8a1), `INST_SME_ENGINE_ST` (0x8a2), `INST_SME_ENGINE_SCALARFP` (0x8a0), and streaming-mode transition events — all readable via kperf with existing privileged-sidecar infrastructure.
- **Why it matters:** ACTIONABLE — on M4+ the matrix-op monitoring gap is closed: kperf can count SME engine retired instructions directly. No new reverse engineering required; our existing kperf sidecar needs only new event codes added to its config for M4+ targets.

### Finding 4: M1 AMX inner-loop microarchitecture characterised — load-issue bound, two on-chip blocks

- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" characterises the M1 AMX inner loop as load-issue bound (not compute-bound) and documents two on-chip AMX blocks; a direct-AMX kernel avoiding the stall achieves 1.17× over Accelerate's fastest BLAS path.
- **Why it matters:** No new kperf events, but confirms the structural reason why AMX activity is hard to isolate from generic CPU counters on M1–M3; reinforces that M4 SME counters (Finding 3) are the right path forward for the main project.

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
