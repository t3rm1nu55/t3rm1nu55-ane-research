# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-01 — sweep (5 findings)

### Finding 1: Orion — first open end-to-end LLM system on ANE, 14 new constraints documented
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 2026
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" (Ramchand Kumaresan) is the first fully open system that compiles and runs LLM workloads directly on the ANE via Apple's private `_ANEClient`/`_ANECompiler` APIs, bypassing CoreML entirely. It catalogs 20 constraints on ANE programs (in MIL IR), of which 14 were previously undocumented, covering memory layout rules, compilation limits, and numerical edge cases.
- **Why it matters:** Constraint catalog is directly useful context for any future ANE telemetry or program-dispatch monitoring in t3rm1nu55-monitorplus; confirms there is still no hardware counter exposed.

### Finding 2: maderix Part 3 and companion ANE GitHub repo — backpropagation on M4 ANE confirmed
- **Source:** maderix Substack / GitHub
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b (Part 3); https://github.com/maderix/ANE (repo)
- **Date:** March 2026
- **Summary:** Part 3 ("Training") documents achieving full forward + backward pass and Adam optimizer updates on a 109M-parameter model and Qwen3-0.6B directly on M4 ANE via `_ANEClient`, `_ANECompiler`, and `_ANEInMemoryModelDescriptor`. The companion GitHub repo has 42 commits and is the only public implementation of ANE backpropagation. Throughput sits at 5–9% of peak capacity, limited by the ANE's compile-and-dispatch model rather than arithmetic throughput. A key constraint: ANE's native SDPA op silently ignores the causal attention mask, requiring decomposition across ANE + CPU.
- **Why it matters:** Confirms that 5–9% observed throughput is not a measurement artefact — the ANE is genuinely underutilised when driven from userspace, and there is no counter surface to observe this; IOReport energy proxy remains the only option.

### Finding 3: ANEForge (2606.17090) and ane-guide (2606.22283) — deepest public ANE documentation yet
- **Source:** arXiv / GitHub (sbryngelson)
- **URL:** https://arxiv.org/abs/2606.17090 (ANEForge); https://arxiv.org/abs/2606.22283 (ane-guide); https://github.com/sbryngelson/ANEForge; https://github.com/sbryngelson/ane-guide
- **Date:** June 12–21, 2026
- **Summary:** Spencer Bryngelson published two complementary works. ANEForge (2606.17090) is a Python library for direct ANE computation without CoreML, compiling a lazy tensor graph (58 fused operators) into a single ANE program with ~90 µs dispatch latency near the 70 µs hardware floor. The companion ane-guide (2606.22283) is a reverse-engineered reference covering the full stack: datapath, roofline, compiler and on-disk program format, weight-compression scheme, kernel driver, firmware, and command protocol, derived from static analysis of Apple's private runtime and direct hardware measurement.
- **Why it matters:** The ane-guide's kernel driver and command protocol documentation is the closest thing yet to a map of what the ANE exposes to the host CPU — worth reading before any future ANE counter probe work.

### Finding 4: M4 and later use ARM SME, not Apple AMX — counter tracking roadmap must bifurcate
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 24, 2026
- **Summary:** "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" (Deyvik Bhan, Georgia Tech) builds a direct-AMX GEMM kernel that outperforms all Accelerate fp32 paths at LLM prefill shapes on M1–M3. Crucially the paper explicitly states that M4 and later chips replaced Apple AMX with ARM's Scalable Matrix Extension (SME), an architecturally distinct coprocessor standardised by ARM.
- **Why it matters:** Any kperf AMX counter discovery is M1–M3 only; M4+ needs a completely separate SME investigation. SME is ARM-standard and may have better PMU event support via ARM's FEAT_PMUv3 extensions — this bifurcation should be tracked explicitly.

### Finding 5: BaseRT reveals per-core Neural Accelerators on M5 GPU via Metal 4 tensor API
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2607.19438
- **Date:** July 2026
- **Summary:** "BaseRT: Advancing Best-in-Class LLM Inference with Apple M5 Neural Accelerators" (Waschkowski et al.) shows that M5 GPUs carry a dedicated Neural Accelerator matrix unit per GPU core, exposed via Apple's new Metal 4 tensor API. Hand-written Metal 4 tensor-core GEMM kernels achieve 6.4× prefill throughput over llama.cpp and 3.9× over MLX on M5 Pro across 15 model configurations.
- **Why it matters:** M5 introduces a third distinct matrix accelerator surface (ANE / AMX-or-SME / GPU Neural Accelerator). The GPU Neural Accelerator is reached through Metal 4 rather than any IOReport or kperf path — a new telemetry gap for t3rm1nu55-monitorplus to track.

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
