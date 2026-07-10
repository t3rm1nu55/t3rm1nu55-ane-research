# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-10 — sweep (3 findings)

### Finding 1: ANEForge — Python library for direct ANE dispatch without CoreML
- **Source:** arXiv + GitHub (sbryngelson/ANEForge)
- **URL:** https://arxiv.org/abs/2606.17090 / https://github.com/sbryngelson/ANEForge
- **Date:** June 12, 2026
- **Summary:** ANEForge compiles a lazy tensor graph (58 fused operators, 19 native bridge operators) into a single ANE program and dispatches it through the same `aned` daemon and kernel-driver stack that CoreML uses internally — without going through CoreML at all. A program call takes ~90µs against the engine's ~70µs dispatch floor; ResNet-18 forward runs end-to-end in 0.33ms. Training (forward + backward + Adam update) runs entirely on ANE. Code is open-source.
- **Why it matters:** ANEForge is the first public library demonstrating direct below-CoreML ANE dispatch. The `aned` IPC/kernel-driver path it uses is exactly where any utilization counter would need to be sampled; the companion architecture paper (Finding 2) reverse-engineers that path in detail.

### Finding 2: "Apple Neural Engine: Architecture, Programming, and Performance" — comprehensive reverse-engineering guide
- **Source:** arXiv (Spencer Bryngelson, same author as ANEForge)
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 2026
- **Summary:** A reverse-engineered guide to ANE internals based on direct measurement on Apple Silicon and static analysis of the private runtime, compiler, kernel driver, and firmware. Documents: the datapath and roofline, the dispatch route below CoreML, the compiler and on-disk program format, the weight-compression scheme, and the kernel driver, firmware, and command protocol.
- **Why it matters:** Most complete public technical account of ANE internals to date. The kernel driver and command protocol sections directly inform what hooks or out-of-band reads might yield utilization metrics without going through Apple's public APIs.

### Finding 3: "Above the Inner Loop" — M1 has two AMX blocks; Accelerate only uses one
- **Source:** arXiv (Deyvik Bhan, Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Characterizes the M1 AMX inner loop as load-issue bound and demonstrates that the M1 chip contains two onchip AMX blocks, only one of which Accelerate uses. A custom direct-AMX kernel that exploits fine multi-thread panels and pre-packed weights exceeds all Accelerate fp32 GEMM paths across 12 LLM prefill shapes, with a 1.58x geometric mean lead.
- **Why it matters:** Any AMX utilization estimate or throughput model that assumes a single AMX block underestimates peak capacity by up to 2x. This is critical context for t3rm1nu55-monitorplus's future AMX utilization inference and for interpreting AMX-related power draw from IOReport.

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
