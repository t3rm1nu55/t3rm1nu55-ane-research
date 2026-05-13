# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-13 — sweep (5 findings)

### Finding 1: maderix ANE Part 3 — training on ANE + new private symbol `_ANEInMemoryModelDescriptor`
- **Source:** maderix Substack + GitHub maderix/ANE
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b | https://github.com/maderix/ANE
- **Date:** March 7, 2026
- **Summary:** Part 3 of the maderix M4 ANE series demonstrates full backpropagation on the Neural Engine using private `_ANEClient`, `_ANECompiler`, and the newly documented `_ANEInMemoryModelDescriptor` APIs. The companion GitHub repo trains up to Qwen3-0.6B (596M params), measuring ~18.6 TOPS FP16 and ~35.1 TOPS INT8 on M4; crucially, INT8 ≈ FP16 throughput because the ANE dequantizes INT8 weights to FP16 before compute, debunking Apple's "38 TOPS" INT8 claim.
- **Why it matters:** `_ANEInMemoryModelDescriptor` is a newly documented private symbol; the INT8/FP16 parity finding corrects any utilization model that treats them as distinct performance bands.

### Finding 2: Orion (arXiv 2603.06728 + mechramc/Orion) — first academic ANE utilization characterization
- **Source:** arXiv + GitHub mechramc/Orion
- **URL:** https://arxiv.org/abs/2603.06728 | https://github.com/mechramc/Orion
- **Date:** ~March 2026
- **Summary:** First academic treatment of ANE at compiler/graph granularity. Key findings: 16–64 op deep graphs achieve 94% ANE utilization; the ANE compiler silently caps each process at ~119 compilations before subsequent calls fail without error; delta compilation cuts recompile latency from ~4,200 ms to ~494 ms (8.5×). The open-source mechramc/Orion runtime ships an ANE utilization-tracking benchmark suite.
- **Why it matters:** The 119-compilation process cap is a hard lifetime constraint for any persistent monitor using `_ANEClient`; the Orion benchmark suite is the first public, reproducible method for measuring ANE utilization as a percentage.

### Finding 3: skyfallsin/apple-neural-engine-field-guide — empirical ANE hardware constraint documentation
- **Source:** GitHub skyfallsin/apple-neural-engine-field-guide
- **URL:** https://github.com/skyfallsin/apple-neural-engine-field-guide
- **Date:** 2026 (tested on macOS 26.3.1 + M3 Max)
- **Summary:** Empirical field guide documenting undocumented ANE hardware constraints and failure modes verified on M3 Max under macOS 26.x: IOSurface minimum width is 32; dispatch overhead follows ~119 µs + bytes/78 GB/s (bandwidth-bound beyond ~5 KB); MIL `tile` op permanently corrupts ANE state for the process lifetime; W-lane batching is the highest-impact throughput optimization.
- **Why it matters:** The dispatch timing formula gives a bandwidth-based ANE utilization estimate from wall-clock measurement alone; the tile-op state-corruption behavior is a silent failure risk for any process that touches ANE.

### Finding 4: arXiv 2604.18788 (NPUMoE) — IOReport energy proxy validated at production LLM scale
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2604.18788
- **Date:** April 20, 2026
- **Summary:** NPUMoE offloads static MoE computation to Apple Silicon ANE, achieving 1.32×–5.55× latency and 1.81×–7.37× energy efficiency improvements. Energy measurement relies on IOReport/powermetrics proxy; the paper independently corroborates that this proxy tracks real ANE load at production-scale LLM workloads across M-series devices.
- **Why it matters:** Provides independent validation that IOReport energy proxy is reliable for ANE load inference at scale; no new counter surface, but methodology confirmation is useful for calibrating t3rm1nu55-monitorplus's ANE power inference.

### Finding 5: macOS Tahoe (26.x) ships — kperf/IOReport compatibility on new major version unverified
- **Source:** Apple Developer Documentation
- **URL:** https://developer.apple.com/documentation/macos-release-notes/macos-26-release-notes
- **Date:** macOS 26.0 shipped Sept 2025; latest 26.5 released May 11, 2026
- **Summary:** Apple shipped macOS Tahoe under a new versioning scheme ("26.x" matching the calendar year), with the largest design overhaul since 2013. No kperf or IOReport breakage was found in release notes or developer forums, but private API compatibility across major versions is historically unguaranteed (cf. macOS 13 powermetrics `bandwidth` removal).
- **Why it matters:** t3rm1nu55-monitorplus depends on private kperf.framework and IOReport; macOS 26.x represents a compatibility risk that needs explicit testing before the project can claim macOS Tahoe support.

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
