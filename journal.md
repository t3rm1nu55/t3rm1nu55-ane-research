# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-21 — sweep (4 findings)

### Finding 1: Orion — First open-source ANE runtime bypassing CoreML entirely
- **Source:** arXiv + GitHub (mechramc)
- **URL:** https://arxiv.org/abs/2603.06728 / https://github.com/mechramc/Orion
- **Date:** March 2026 (missed by initial seed)
- **Summary:** Orion is an end-to-end LLM training and inference runtime for the Apple Neural Engine that uses `_ANEClient` and `_ANECompiler` private APIs directly, without CoreML. The paper reports 94% ANE utilization at 32+ layer depth (measured via framework-level scheduling metrics, not hardware counters) and 6.6 TFLOPS/W peak efficiency on M4 Max. The Objective-C source is MIT-licensed and publicly available. Orion explicitly builds on the maderix private-API reverse-engineering and extends it to production quality: compiler pipeline, stable training, checkpoint resume, and a benchmark harness.
- **Why it matters:** mechramc/Orion is now the definitive public reference implementation of `_ANEClient`/`_ANECompiler` usage; the utilization metric it reports (graph-depth scheduling saturation) is the closest anyone has come to a public ANE occupancy signal.

### Finding 2: maderix/ANE GitHub repo — companion code for full reverse-engineering series
- **Source:** GitHub (maderix) + maderix Substack Parts 1 & 3
- **URL:** https://github.com/maderix/ANE / https://maderix.substack.com/p/inside-the-m4-apple-neural-engine / https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** February–March 2026 (missed by initial seed)
- **Summary:** The seed tracked the Part 2 benchmarks post but missed Parts 1 (full software-stack reverse engineering from CoreML down to IOKit driver) and Part 3 (training a 109M-parameter transformer on the ANE with full forward/backward pass and Adam). The companion GitHub repo was also not tracked and contains working Objective-C for `_ANEClient` scheduling and memory layout, plus Python weight-conversion scripts.
- **Why it matters:** Parts 1 and 3 add critical stack detail — Part 1 documents how to enumerate and compile to the ANE without CoreML, Part 3 proves the inference-only hardware can sustain training gradients — and the GitHub code gives concrete `_ANEClient` call sequences to reference.

### Finding 3: arXiv:2604.18788 — NPUMoE confirms ANE utilization remains inferred, not measured
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2604.18788
- **Date:** April 20, 2026
- **Summary:** NPUMoE offloads dense expert computation in Mixture-of-Experts LLMs to the Apple Silicon ANE, achieving 1.32×–5.55× latency reduction and 1.81×–7.37× energy efficiency gains. Expert capacity and routing are handled via offline calibration rather than real-time counter feedback, confirming that no NPU counter API is used. The paper was published after the last sweep.
- **Why it matters:** Provides independent confirmation (post-April 7) that the field is still doing capacity estimation via calibration rather than hardware counters — the open problem documented in the seed remains unsolved.

### Finding 4: SiliconScope v2.1.3 — new IOReport channel reference, released today
- **Source:** GitHub (kennss)
- **URL:** https://github.com/kennss/SiliconScope
- **Date:** June 21, 2026 (released today)
- **Summary:** SiliconScope is a native SwiftUI Apple Silicon monitor with ANE, Media Engine, and memory-bandwidth tracking via unsandboxed IOReport and SMC access. Version 2.1.3 released today; the repo includes `docs/ioreport-channels.md` with a verified channel map for M1 Max on macOS 26.5. Key channels documented: `ANE0`/`ANE1` (Energy Model group, power only, "0 when idle"), ProRes/codec bandwidth (`PROSES / STRM CODEC DCS`), per-cluster CPU power (`EACC_CPU`, `PACC0_CPU`, `PACC1_CPU`). The project explicitly labels ANE usage as a "power-normalized estimate" because Apple does not expose ANE occupancy.
- **Why it matters:** `docs/ioreport-channels.md` is a fresh, chip-verified IOReport channel listing to compare against macmon and socpowerbud for any gaps; the project's honest labeling of ANE estimation is a useful signal that no hidden occupancy channel has appeared.

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
