# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-18 — sweep (4 findings)

### Finding 1: "Apple Neural Engine: Architecture, Programming, and Performance" (arXiv 2606.22283)
- **Source:** arXiv / Spencer H. Bryngelson, Georgia Institute of Technology
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 21, 2026
- **Summary:** The most comprehensive public reverse-engineering of the ANE to date, covering A11–A18 and M1–M5 families. Documents the datapath, roofline throughput bounds, dispatch route below CoreML, on-disk compiler format, weight-compression scheme, and—critically—the kernel driver, firmware, and command protocol. Based on direct measurement on M1 and M5 hardware plus static analysis of the private runtime. Web edition at ane-guide.readthedocs.io; companion GitHub at sbryngelson/ane-guide.
- **Why it matters:** The command-protocol and kernel-driver documentation may contain performance-monitoring register definitions or telemetry hooks that could underpin an ANE utilization metric in t3rm1nu55-monitorplus. First stop for anyone trying to expose hardware-level ANE telemetry.

### Finding 2: ANEForge — Python direct ANE dispatch (arXiv 2606.17090 / sbryngelson/ANEForge)
- **Source:** arXiv / GitHub
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 12, 2026
- **Summary:** A Python library enabling compilation and execution of neural-network programs directly on the ANE, bypassing CoreML entirely via Apple's private `_ANEClient`/`_ANECompiler` stack. Exposes 19 native ANE operations CoreML cannot reach (including fused attention), supports training (forward + backward + Adam), and handles int8/int4 weight compression. Does not expose hardware performance counters; energy is measured via external `powermetrics`.
- **Why it matters:** Establishes the direct-dispatch API surface cleanly in open code. Rust FFI bindings to the same private framework stack could give t3rm1nu55-monitorplus a direct ANE command path; power can already be cross-referenced from IOReport.

### Finding 3: maderix/ANE — transformer training on ANE via private APIs
- **Source:** GitHub (maderix/ANE)
- **URL:** https://github.com/maderix/ANE
- **Date:** Active March 2026 (not captured in initial seed)
- **Summary:** Implements from-scratch transformer training on the ANE using `_ANEClient`, `_ANECompiler`, and `_ANEInMemoryModelDescriptor`. Includes `sram_bench.m` and `sram_probe.m` for SRAM bandwidth probing; reports peak throughput at 18.6 TFLOPS FP16 / 35.1 TOPS INT8 W8A8 on M4, with current training runs achieving ~5–9% of peak TOPS. No direct hardware-counter utilization API, but the SRAM probing scripts are novel instrumentation.
- **Why it matters:** The SRAM probe methodology is the closest public work to instrumenting ANE internals. The throughput figures update the best known public characterization of M4 ANE (previously 19 TFLOPS from maderix's 2024 Substack; now confirmed with direct dispatch code).

### Finding 4: "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" (arXiv 2606.25426)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 24, 2026
- **Summary:** Detailed microarchitectural characterization of Apple M1 AMX: the M1 SoC contains two independent AMX blocks per CPU cluster; the inner loop is load-issue bound and achieves only ~610–680 GFLOPS single-thread (under half the load-free theoretical rate); gains above Accelerate come from fine multi-thread panel sizing to keep both AMX blocks fed. Measurement methodology is micro-benchmarking, not hardware PMU counters.
- **Why it matters:** Confirms the M1 dual-AMX-block structure and identifies the binding resource (load bandwidth, not compute). This is the reference to cite when designing AMX utilization inference: load-bandwidth saturation is the better proxy than FMA issue rate.

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
