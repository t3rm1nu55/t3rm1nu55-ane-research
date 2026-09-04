# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-09-04 — sweep (4 findings)

### Finding 1: ANEForge — Python library for direct ANE computation without CoreML
- **Source:** arXiv + GitHub (sbryngelson/ANEForge)
- **URL:** https://arxiv.org/abs/2606.17090 · https://github.com/sbryngelson/ANEForge
- **Date:** 2026-06-12
- **Summary:** ANEForge compiles a lazy Python tensor graph (58 fused operators + 19 native bridge operators) into a single ANE program dispatched through the same daemon/kernel-driver stack Apple uses internally, bypassing CoreML entirely. The forward pass, backward pass, and Adam optimizer update all execute as native ANE programs; the library also supports LLM decode/prefill with cross-step KV cache, ONNX frontend, and int8/int4 quantized weights. It is published on PyPI (`aneforge`) and includes a companion Hugging Face org.
- **Why it matters:** The most complete public ANE programming surface yet. The library's dispatch route through the ANE daemon provides a concrete reference for where to intercept utilization signals in t3rm1nu55-monitorplus.

### Finding 2: "Apple Neural Engine: Architecture, Programming, and Performance" — comprehensive RE reference
- **Source:** arXiv + GitHub (sbryngelson/ane-guide)
- **URL:** https://arxiv.org/abs/2606.22283 · https://github.com/sbryngelson/ane-guide · https://ane-guide.readthedocs.io
- **Date:** 2026-06-21
- **Summary:** Spencer Bryngelson's paper (and companion web guide) reverse-engineers the full ANE stack from A11 to M5: datapath and roofline, dispatch route below CoreML, compiler and on-disk E5 program format, weight-compression scheme, and the kernel driver, firmware, and command protocol beneath them. The methodology combines direct measurement on Apple Silicon with static analysis of the private runtime, compiler, kernel driver, and firmware. This is the most comprehensive public reference on ANE internals ever published.
- **Why it matters:** The kernel driver and firmware command protocol sections are directly relevant to implementing ANE activity detection in t3rm1nu55-monitorplus — this is the reference document the field has been missing.

### Finding 3: BaseRT reveals M5 "Neural Accelerators" exposed via Metal 4 tensor API
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2607.19438
- **Date:** 2026-07-21
- **Summary:** Apple's M5 generation introduces dedicated on-die Neural Accelerators — distinct from prior ANE — exposed through a new Metal 4 tensor API. BaseRT is a native Metal inference runtime with hand-written Metal 4 tensor-core kernels that route matrix multiplications through these units, achieving up to 6.4× over llama.cpp and 2.1× over MLX at 2048-token prompts on M5 Pro. The existence of a (semi-)public Metal 4 API for accessing these units is architecturally significant.
- **Why it matters:** On M5+ the ANE-monitoring problem partially reframes: Metal 4 may expose performance counter hooks through the standard GPU counter infrastructure that are inaccessible for previous-generation ANE. t3rm1nu55-monitorplus should investigate Metal 4 GPUCounters for Neural Accelerator activity on M5.

### Finding 4: macmon v0.8.1 restores per-core metrics for M3 Ultra
- **Source:** GitHub (vladkens/macmon)
- **URL:** https://github.com/vladkens/macmon/commit/6919d7781b6c55a6e3bedff83a210435837e1dfe
- **Date:** 2026-08-04
- **Summary:** vladkens/macmon commit 6919d77 fixes per-core IOReport metrics for M3 Ultra, which had apparently regressed. The surrounding v0.8.1 release also refactors the codebase into separate `app` and `lib` folders, making the IOReport-reading library code more independently importable.
- **Why it matters:** We vendor the IOReport access pattern from macmon; this fix confirms M3 Ultra has a distinct channel layout that requires separate handling. The lib/app split makes it easier to track just the IOReport code changes.

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
