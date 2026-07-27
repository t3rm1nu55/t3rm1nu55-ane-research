# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-27 — sweep (4 findings)

### Finding 1: Spencer Bryngelson publishes comprehensive reverse-engineered ANE reference (ane-guide + arXiv)
- **Source:** arXiv 2606.22283 / github.com/sbryngelson/ane-guide
- **URL:** https://arxiv.org/abs/2606.22283 · https://ane-guide.readthedocs.io
- **Date:** 2026-06-21
- **Summary:** Georgia Tech researcher Spencer Bryngelson published the most thorough public treatment of ANE internals to date, covering: the compute datapath and roofline, the full dispatch route below CoreML (daemon → IOKit driver → firmware), the compiled program format, weight-compression scheme, and the command protocol between the driver and engine. The work is grounded in direct measurement on hardware and static decompilation of Apple's private runtime. An accompanying web edition and GitHub repo make it a living reference.
- **Why it matters:** The kernel driver and command protocol chapters are the first public attempt to describe how software communicates with the ANE at the register/command level — the exact layer where any hardware counter would be exposed.

### Finding 2: ANEForge — Python dispatch directly to ANE without CoreML
- **Source:** arXiv 2606.17090 / github.com/sbryngelson/ANEForge
- **URL:** https://arxiv.org/abs/2606.17090 · https://github.com/sbryngelson/ANEForge
- **Date:** 2026-06-12
- **Summary:** Same author as Finding 1. ANEForge is a Python package that compiles a lazy tensor graph (58 fused operators) into an ANE program and dispatches it through the same `_ANEClient` daemon and kernel-driver stack Apple's own frameworks use, bypassing CoreML entirely. A pre-trained ResNet-18 forward pass completes in 0.33 ms; round-trip dispatch overhead is ~90 µs, near the 70 µs hardware floor. Notably, it also runs training (forward + backward + optimizer update) on the ANE.
- **Why it matters:** Working code that talks directly to the ANE IOKit driver. Inspecting ANEForge's dispatch path is the most concrete starting point for discovering whether timing or counter registers are readable from the host side.

### Finding 3: maderix/ANE — transformer training via private ANE APIs; claims ANE utilization measurement
- **Source:** github.com/maderix/ANE (new repository, follow-up to previously tracked Substack)
- **URL:** https://github.com/maderix/ANE
- **Date:** 2026 (exact date not confirmed)
- **Summary:** maderix has open-sourced a from-scratch transformer training implementation running on ANE via `_ANEClient` / `_ANECompiler` private APIs and the MIL (Model Intermediate Language) format. The README reports "11.2% ANE utilization on M4" for a benchmark workload. Skeptical note: this utilization figure is most likely derived from IOReport Energy Model power sampling (observed power / peak power), not a true hardware counter — the 11.2% figure and the ~5–9% range cited elsewhere are consistent with energy-ratio inference, not a cycle-count metric.
- **Why it matters:** Adds a code-level reference implementation of `_ANEClient` usage on M4. The utilization claim (and its likely methodology) is worth examining to see if they surface any previously unlogged IOReport channel names for ANE.

### Finding 4: M5 ships Neural Accelerators in every GPU core, exposed via Metal 4 tensor API
- **Source:** arXiv 2607.19438 (BaseRT paper)
- **URL:** https://arxiv.org/abs/2607.19438
- **Date:** 2026-07-21
- **Summary:** Apple's M5 GPU architecture introduces per-GPU-core Neural Accelerators — on-die matrix units distinct from the ANE — exposed through the public Metal 4 tensor API. BaseRT, a native Metal LLM runtime, exploits these to achieve up to 6.4× the prompt-processing throughput of llama.cpp on M5 Pro. Unlike the ANE, Metal has proper public profiling infrastructure (Metal Performance Shaders counters, GPU Frame Capture).
- **Why it matters:** For M5 hardware, there is now a *public* API surface (Metal 4 tensor ops + Metal GPU counters) that may expose Neural Accelerator utilization without private API hacking. t3rm1nu55-monitorplus should plan a Metal counter path alongside the existing IOReport/kperf paths for M5+ systems.

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
