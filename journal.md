# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-03 — sweep (4 findings)

### Finding 1: Comprehensive ANE architecture RE: datapath, kernel driver, and command protocol (A11–M5)
- **Source:** arXiv 2606.22283 / sbryngelson/ane-guide
- **URL:** https://arxiv.org/abs/2606.22283 | https://ane-guide.readthedocs.io/en/latest/ | https://github.com/sbryngelson/ane-guide
- **Date:** June 21, 2026
- **Summary:** Spencer Bryngelson (Georgia Tech) published a reverse-engineered account of the Apple Neural Engine covering A11 through M5, based on direct measurement and static analysis of the private runtime and kernel. It documents the datapath and roofline, dispatch route below CoreML, compiler and on-disk program format (E5 binary), weight-compression scheme, and — most significantly — the kernel driver, firmware, and command protocol. Companion repos: sbryngelson/ane-guide and sbryngelson/ANEForge (Python framework for direct ANE dispatch).
- **Why it matters:** The kernel driver and command protocol documentation is the most thorough public account of ANE internals to date; reviewing it may reveal internal state-reporting registers or telemetry hooks usable as ANE utilization proxies.

### Finding 2: INST_SME_ENGINE_* counters appear in M4 kpep database; M4+ transitioned from Apple AMX to ARM SME
- **Source:** jiegec/apple-pmu (as4.md) + arXiv 2606.25426 (Bhan, Georgia Tech)
- **URL:** https://github.com/jiegec/apple-pmu/blob/master/as4.md | https://arxiv.org/abs/2606.25426
- **Date:** 2026 (repo ongoing); June 24, 2026 (arXiv paper)
- **Summary:** The jiegec/apple-pmu repo documents that the M4 kpep database (as4.plist) introduced `INST_SME_ENGINE_*` counter events, while M5 (as5) adds `LD_SRC_*` and PL2 cache events. Separately, arXiv 2606.25426 (Bhan, June 2026) confirms M1–M3 use Apple's proprietary AMX coprocessor while M4+ switched to the standard ARM Scalable Matrix Extension (SME), which has PMU counter events defined in the ARM architecture specification.
- **Why it matters:** Directly actionable — `INST_SME_ENGINE_*` events in the M4 kpep database may be readable via kperf today, giving the first hardware-counter proxy for matrix coprocessor utilization on M4+ chips.

### Finding 3: maderix "Inside the M4 ANE" 2026 series (Parts 1–3) — full IOKit driver mapping
- **Source:** maderix Substack (new 2026 multi-part series, distinct from the 2024 post tracked in references.md)
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine | https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** March 2, 2026 (Part 1)
- **Summary:** A new multi-part Substack series maps the full ANE software stack from CoreML down to IOKit: 40+ private classes including `_ANEClient`, `_ANECompiler`, and `_ANEInMemoryModelDescriptor`; E5 binary format; and the IOSurface protocol. Part 3 covers full ANE training (backpropagation). Accompanied by the open-source maderix/ANE GitHub repo demonstrating Stories110M training at 91ms/step and INT8 W8A8 quantization at 1.88x throughput.
- **Why it matters:** The deep-dive into IOKit private class hierarchy and in-memory dispatch path is the most detailed public account of ANE driver mechanics and identifies "unexplored territory" sections that may include internal telemetry endpoints.

### Finding 4: Orion — first open ANE LLM training system; catalogs 20 ANE compiler restrictions
- **Source:** arXiv 2603.06728 / mechramc/Orion GitHub
- **URL:** https://arxiv.org/abs/2603.06728 | https://github.com/mechramc/Orion
- **Date:** March 6, 2026
- **Summary:** Orion is the first open end-to-end system for LLM training and inference directly on the ANE, bypassing CoreML entirely via `_ANEClient` and `_ANECompiler`. The paper catalogs 20 documented ANE compiler constraints (operation/shape restrictions). Training Stories110M at 91ms/step is demonstrated alongside a compiler pipeline and checkpoint resume.
- **Why it matters:** The catalog of 20 ANE restrictions is the most comprehensive public documentation of ANE operational boundaries; understanding these constraints is prerequisite to any future ANE dispatch-based utilization benchmarking in t3rm1nu55-monitorplus.

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
