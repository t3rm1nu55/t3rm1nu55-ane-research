# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-03 — sweep (3 findings)

### Finding 1: jiegec/apple-pmu — M4 and M5 kpep dumps include ARM SME engine counter events
- **Source:** [jiegec/apple-pmu](https://github.com/jiegec/apple-pmu)
- **URL:** https://github.com/jiegec/apple-pmu
- **Date:** Repo ongoing; as4.md (M4) added ~2024; as5.md (M5) added after March 2026 M5 launch
- **Summary:** This untracked repo dumps `/usr/share/kpep` plist files for every Apple Silicon generation into readable Markdown. M4 (as4.md, 105 events) and M5 (as5.md, 136 events) both contain ARM SME engine counters absent from M1–M3: `INST_SME_ENGINE_ALU` (retired non-load/store SME instructions), `CORE_WAITING_SME_ENGINE_CYCLE` (stall cycles waiting on SME engine), `SME_ENGINE_SM_ENABLE` / `SME_ENGINE_ZA_ENABLE` (streaming mode transitions). Since Apple replaced proprietary AMX with standard ARM SME starting with M4, these events are the first hardware counter signal for matrix coprocessor activity on any Apple Silicon chip. The M5 as5.md adds 31 additional events and shows continued SME counter coverage.
- **Why it matters:** `CORE_WAITING_SME_ENGINE_CYCLE` and `INST_SME_ENGINE_ALU` are likely samplelable via the kperf sidecar on M4/M5 Macs — this could be the first real counter-based matrix utilization metric for t3rm1nu55-monitorplus (M4+ only; M1–M3 remain opaque).

### Finding 2: arXiv 2604.18788 — NPUMoE: Efficient MoE LLM Inference on Apple Silicon NPUs
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2604.18788
- **Date:** April 20, 2026
- **Summary:** NPUMoE is a new inference runtime that offloads dense, shape-static MoE subgraphs to the ANE while routing dynamic expert selection to CPU/GPU. The paper characterizes ANE operational limits in detail: shape-specific execution constraints, incompatibility with top-k/scatter ops, high per-kernel dispatch overhead, and a ceiling of ~127 simultaneous evaluation requests. Measurement methodology relies on wall-clock throughput and powermetrics energy, not hardware counters.
- **Why it matters:** Confirms the ANE has no counter-based utilization path and documents dispatch-overhead numbers that would inform any future profiling approach; the ~127 in-flight request ceiling is a new architectural constant worth noting.

### Finding 3: maderix Part 3 — Training on the M4 ANE (missed in initial seed)
- **Source:** maderix Substack
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** March 7, 2026 (published before last sweep; not captured in seed)
- **Summary:** Third installment of the maderix ANE series demonstrates stable neural network training (not just inference) on the M4 ANE via `_ANEClient`/`_ANECompiler` private APIs. Training achieves only 5–9% of peak throughput due to weight-reload overhead; 91 ms/step for a 110 M-parameter transformer. No new hardware counters discovered; the companion `maderix/ANE` GitHub repo provides working code.
- **Why it matters:** Closes out the maderix series; confirms the private API surface (`_ANEClient`, `_ANECompiler`, `_ANEInMemoryModelDescriptor`) is stable enough for training workloads, but low utilization during training makes power-based utilization inference even noisier for non-inference workloads.

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
