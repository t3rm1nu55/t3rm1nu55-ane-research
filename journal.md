# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-10 — sweep (6 findings)

### Finding 1: Bryngelson ANE Architecture paper — deepest public hardware doc to date
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 21, 2026
- **Summary:** Spencer Bryngelson (Georgia Tech) published the first comprehensive reverse-engineered account of ANE hardware from A11 through M5, covering the internal memory hierarchy, the 32 MB on-chip SRAM throughput cliff, and full documentation of the ANE command format and IOKit driver interface. Derives from direct measurement on M1/M5 silicon plus static decompilation of the private runtime, compiler, kernel driver, and firmware. Companion repo: [sbryngelson/ane-guide](https://github.com/sbryngelson/ane-guide) with ReadTheDocs edition.
- **Why it matters:** The IOKit driver interface documentation is the closest thing yet to a roadmap for exposing ANE utilization from host code; worth auditing for any counter or telemetry surface the driver exposes.

### Finding 2: ANEForge — first open-source direct ANE access path with utilization measurement
- **Source:** arXiv + GitHub + PyPI
- **URL:** https://arxiv.org/abs/2606.17090 / https://github.com/sbryngelson/ANEForge
- **Date:** June 12, 2026
- **Summary:** Companion library to Finding 1 by the same author. ANEForge compiles a lazy Python tensor graph (58 fused operators) directly into an ANE program, bypassing CoreML entirely. Reports 11.2% ANE utilization for a single transformer layer (dim=768, seq=512) on M4 and 94% utilization for deep 16–64 op graphs — the first programmatic, publicly documented path to drive the ANE from user code with utilization data as a side effect. Available on PyPI as `aneforge`.
- **Why it matters:** Directly actionable: ANEForge's bypass path and the IOKit interface it uses may be the mechanism to expose real ANE utilization to t3rm1nu55-monitorplus.

### Finding 3: AMX→ARM SME transition confirmed for M4+
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 24, 2026
- **Summary:** Microbenchmarks characterize the M1 AMX inner loop as load-issue bound on the AMX FMA32/load issue port, with single-thread throughput constrained to 610–680 GFLOPS. Critically, the paper confirms that M4 and later switch from Apple's proprietary AMX coprocessor to ARM SME (Scalable Matrix Extension), meaning the AMX microarchitecture as characterized applies to M1–M3 only. First peer-reviewed quantitative characterization of the AMX load-issue port constraint.
- **Why it matters:** The project's AMX tracking scope must be split by chip generation — AMX strategy for M1–M3, ARM SME strategy for M4+. SME has different (and partially public) ISA documentation.

### Finding 4: BaseRT reveals M5 GPU per-core Neural Accelerator — new architecture
- **Source:** arXiv / ResearchGate
- **URL:** https://arxiv.org/abs/2607.19438 / https://www.researchgate.net/publication/410721363
- **Date:** July 21, 2026
- **Summary:** BaseCompute's second BaseRT paper reveals that every M5 GPU core carries a dedicated on-die Neural Accelerator exposed through the Metal 4 tensor API, achieving up to 6.4× higher prefill throughput than llama.cpp. This is the first public description of the M5 GPU architecture differing fundamentally from M4 by embedding per-core matrix units. Note: distinct from the earlier "BaseRT: Best-in-Class LLM Inference on Apple Silicon via Native Metal" (ResearchGate 408340817, ~July 1, 2026).
- **Why it matters:** M5 introduces a new compute unit (GPU-embedded Neural Accelerators) that is not the ANE proper; any ANE counter work needs to track whether Apple exposes this unit separately or routes it through the same ANE IOKit interface.

### Finding 5: macmon Frida research documents powermetrics' exact IOReport subscription structure
- **Source:** vladkens/macmon commit 40f4e46
- **URL:** https://github.com/vladkens/macmon/commit/40f4e46408c20e63cd51b1b3416c170ac8f447ea
- **Date:** July 23, 2026
- **Summary:** macmon added a `research/powermetrics/` directory containing Frida scripts that intercept IOReport API calls in `/usr/bin/powermetrics`. Reveals that powermetrics makes exactly three IOReport subscriptions per cycle: "CPU Complex Performance States" (4 channels: ECPU/ECPM/PCPU/PCPM histograms), "CPU Core Performance States" (8 channels, one per core), and "Energy Model" (136 channels covering CPU, GPU, ANE, DRAM, display, media, SRAM). Each subscription is sampled once per cycle; the prior macmon multi-sample approach was a presentation artefact, not matching powermetrics' methodology.
- **Why it matters:** Confirms that the "Energy Model" subscription carries ANE energy in 136 channels and is sampled once per cycle; directly informs how we structure IOReport subscriptions and sampling cadence in the Rust codebase.

### Finding 6: macmon exposes active residency as a distinct metric; documents GPU IOReport channel
- **Source:** vladkens/macmon commit 3010f1fb
- **URL:** https://github.com/vladkens/macmon/commit/3010f1fb7e209c334a1f948ea4386d92e1d761d2
- **Date:** June 9, 2026
- **Summary:** macmon v0.8 adds `active_ratio` (raw IOReport active residency, no frequency weighting) as a distinct field from `usage_ratio` (frequency-scaled effective usage) across CPU clusters, per-core, and GPU. The GPU metric is sourced from IOReport group `"GPU Stats"`, subgroup `GPU_FREQ_DICE_SUBG`, channel `"GPUPH"`. New Prometheus metrics: `macmon_cpu_active_ratio`, `macmon_ecpu_active_ratio`, `macmon_pcpu_active_ratio`, `macmon_gpu_active_ratio`.
- **Why it matters:** The active/frequency-scaled distinction is one t3rm1nu55-monitorplus should expose; the GPU channel name `GPUPH` and group `GPU Stats` are the authoritative IOReport identifiers we should be using.

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
