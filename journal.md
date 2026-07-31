# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-31 — sweep (6 findings)

### Finding 1: ane-guide — complete ANE architecture reference, A11–M5 (arXiv:2606.22283)
- **Source:** arXiv / GitHub sbryngelson/ane-guide
- **URL:** https://arxiv.org/abs/2606.22283 · https://github.com/sbryngelson/ane-guide · https://ane-guide.readthedocs.io
- **Date:** June 2026
- **Summary:** Spencer Bryngelson (Georgia Tech) published the most comprehensive public ANE reference to date, covering A11–M5. It documents the full dispatch stack below CoreML: the `aned` daemon, the compiler and MIL program format, the weight-compression scheme, the IOKit kernel driver, firmware, and command protocol, with per-chip roofline tables from direct measurement on M1 and M5.
- **Why it matters:** The kernel driver and command-protocol sections are the closest any public document has come to describing what sits between the host CPU and ANE hardware registers — the likely path to any future utilization counter exposure.

### Finding 2: ANEForge — working Python library for direct ANE dispatch (arXiv:2606.17090)
- **Source:** arXiv / GitHub sbryngelson/ANEForge
- **URL:** https://arxiv.org/abs/2606.17090 · https://github.com/sbryngelson/ANEForge
- **Date:** June 12, 2026
- **Summary:** Same author as ane-guide. ANEForge is a Python package that compiles a lazy tensor graph (58 fused operators, 19 native bridge operators) into an ANE program and dispatches it through the private `aned` daemon stack, bypassing CoreML entirely. It supports inference, fused attention, int8/int4/sparse weights, and full training (forward + backward + optimizer on-chip).
- **Why it matters:** Provides working code exercising the private dispatch path documented in ane-guide; confirms `aned` is the reachable interface below CoreML, which is relevant to any future ANE utilization hook.

### Finding 3: maderix Part 3 — first ANE training via reverse-engineered private APIs (Substack / GitHub)
- **Source:** maderix Substack (Part 3) / GitHub maderix/ANE
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b · https://github.com/maderix/ANE
- **Date:** March 2026 (not captured in initial seed)
- **Summary:** maderix (Manjeet Singh) published Part 3 of the M4 ANE series, demonstrating full backpropagation on ANE via reverse-engineered `_ANEClient` and `_ANECompiler` private APIs — no CoreML, Metal, or GPU. Working code on GitHub trained a 109M-parameter model from scratch, then scaled to Qwen3-0.6B (596M params). The `api_exploration.m` file catalogs 40+ private classes including `_ANEInMemoryModelDescriptor`.
- **Why it matters:** The `_ANEClient` usage patterns in this codebase are the most complete public example of interacting with the ANE kernel driver; the private class catalog is a useful complement to ane-guide.

### Finding 4: AMX microarchitecture — M1 has two AMX blocks, load-issue bound (arXiv:2606.25426)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Deyvik Bhan (Georgia Tech) characterizes the M1 AMX at the microarchitectural level: the M1 has two on-chip AMX blocks (filling the second requires fine multi-thread panels), the M1 AMX is load-issue bound at ~610–680 GFLOPS single-thread, and a custom kernel exploiting both blocks + pre-packed weights beats Accelerate by 1.17–1.58× across all 12 LLM prefill GEMM shapes.
- **Why it matters:** The two-AMX-block discovery is previously undocumented public knowledge; combined with the load-issue-bound characterization, this informs what a hypothetical AMX utilization counter would measure. No AMX hardware counters were found, but the paper establishes the performance model.

### Finding 5: SiliconScope — new IOReport tool exposing ANE + Media Engine + bandwidth
- **Source:** GitHub kennss/SiliconScope
- **URL:** https://github.com/kennss/SiliconScope
- **Date:** 2026 (v2.3.0 adds M4 Pro/Max GPU; v2.4.0 adds menu-bar improvements)
- **Summary:** SiliconScope is a sudoless SwiftUI Apple Silicon monitor exposing ANE power, Media Engine power, and unified-memory bandwidth with CPU/GPU/Media split — channels beyond what macmon currently surfaces. It cites NeoAsitop and SocPowerBuddy for IOReport/SMC channel knowledge and adds a live LLM "bandwidth-bound vs compute-bound" verdict.
- **Why it matters:** Potential source of IOReport channel names for ANE and Media Engine power that t3rm1nu55-monitorplus may be missing; worth inspecting their IOReport subscription code for new channel identifiers.

### Finding 6: BaseRT — M5 GPU ships dedicated on-die Neural Accelerators via Metal 4 tensor API (arXiv:2607.19438)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2607.19438
- **Date:** July 21, 2026
- **Summary:** Waschkowski, Rathnayaka, Wesemann describe M5's GPU architecture: every GPU core carries a dedicated "Neural Accelerator" (on-die matrix unit) exposed through the Metal 4 tensor API. BaseRT implements hand-written Metal 4 tensor-core kernels that route matrix multiplications through these units, achieving 6.4× higher prompt throughput than llama.cpp on M5 Pro across 15 model configurations.
- **Why it matters:** M5 introduces a new GPU-side matrix unit exposed via Metal 4 — distinct from ANE and AMX — that may have its own IOReport or Metal performance counter surface worth tracking in future chip generations.

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
