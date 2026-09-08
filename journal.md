# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-09-08 — sweep (5 findings)

### Finding 1: ANE kernel driver and command protocol fully documented (arxiv:2606.22283)
- **Source:** arXiv / Spencer H. Bryngelson (Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.22283 — companion guide: https://ane-guide.readthedocs.io/
- **Date:** 2026-06-21
- **Summary:** The most comprehensive public reverse-engineering of the ANE to date. Documents the datapath, roofline throughput/energy bounds, the dispatch route below Core ML (bypassing `_ANEClient`), compiler and on-disk program format, weight compression, kernel driver, firmware, and IOKit command protocol. Based on direct hardware measurement and static analysis of the private runtime. Companion GitHub repo: `sbryngelson/ane-guide`.
- **Why it matters:** The kernel driver and firmware command protocol sections are the first public documentation of the IOKit surface the ANE exposes — potentially the path to reading any internal counters that exist.

### Finding 2: ANE memory-controller byte counters readable during inference (arxiv:2608.22110)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2608.22110
- **Date:** 2026-08 (exact date TBD)
- **Summary:** Measurement study of LLM placement and decode speed on the ANE. Key methodology: reads the ANE's **memory-controller byte counters** during inference to establish what actually executed on-chip rather than trusting compiler intent. Finds that placement is a property of computation _expression_, not semantics — a fused RMSNorm runs on ANE while an arithmetically identical decomposition is CPU-only.
- **Why it matters:** First published use of ANE hardware memory-controller counters as a utilization signal. The counter-read technique is directly relevant to t3rm1nu55-monitorplus's open problem of measuring ANE utilization without a public API.

### Finding 3: maderix/ANE — from-scratch transformer training via reverse-engineered `_ANEClient` (GitHub)
- **Source:** GitHub — maderix/ANE
- **URL:** https://github.com/maderix/ANE
- **Date:** 2026 (active)
- **Summary:** Open repo demonstrating full forward+backward pass transformer training on ANE via direct `_ANEClient`/`_ANECompiler` private API calls, bypassing Core ML. Key architectural finding: expressing matmul as a 1×1 convolution yields 3× higher throughput than the ANE's native matmul path because convolution is the primary compute primitive. Includes a follow-up repo (`maderix/h3.c-ane`) that runs H3 video DiT blocks as raw MIL graphs.
- **Why it matters:** Extends the private API surface map beyond what `hollance/neural-engine` covers; the conv1×1 throughput finding reveals that throughput measurement depends on operation encoding, complicating any utilization model.

### Finding 4: AMX on M1 is load-issue bound — new microarchitectural detail (arxiv:2606.25426)
- **Source:** arXiv — Deyvik Bhan
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** 2026-06
- **Summary:** Direct-AMX GEMM kernel that exceeds Accelerate's fastest path (BNNS Graph) by 1.17× at all 12 LLM prefill shapes by exploiting two M1-specific levers: fine multi-thread panels that fill the second on-chip AMX block, and pre-packed weights. Core microarchitectural finding: the AMX inner loop is **load-issue bound** — any operand load interleaved with the FMA32 stream drops single-thread throughput from ~1.4 TFLOPS to a ~610–680 GFLOPS band.
- **Why it matters:** Confirms AMX has two on-chip blocks (relevant for per-core counter design) and establishes that load-issue is the binding constraint, not compute — any future AMX counter model should track memory bandwidth, not FLOPS.

### Finding 5: vladkens/macmon exposes IOReport active residency ratios (v0.8.0)
- **Source:** GitHub — vladkens/macmon
- **URL:** https://github.com/vladkens/macmon/commit/3010f1fb7e209c334a1f948ea4386d92e1d761d2
- **Date:** 2026-06-09
- **Summary:** macmon v0.8.0 added `feat: expose active residency ratios` via IOReport (commit `3010f1fb`), plus per-core M3 Ultra metric fixes and IOReport interval correctness fixes in v0.8.1. The residency ratios are a per-cluster metric derived from IOReport "CPU Stats" / "GPU Stats" channels.
- **Why it matters:** Active residency ratios are a new IOReport-derived metric that t3rm1nu55-monitorplus should consider adopting; also confirms the "CPU Stats" channel survives into macOS 26.x-era firmware.

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
