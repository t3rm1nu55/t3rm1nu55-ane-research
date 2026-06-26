# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-26 — sweep (2 findings)

### Finding 1: M5 kpep exposes SME engine hardware counters via kperf

- **Source:** jiegec/apple-pmu
- **URL:** https://github.com/jiegec/apple-pmu/blob/master/as5.md
- **Date:** Available since M5 MacBook Pro shipped (March 2026); M5 kpep files documented in repo
- **Summary:** The M5 (AS5) kpep database — macOS's naming layer for kperf PMU events — includes 12 new SME engine counters: `INST_SME_ENGINE_ALU`, `INST_SME_ENGINE_LD/ST`, `INST_SME_ENGINE_SCALARFP`, `INST_SME_ENGINE_PACKING_FUSED`, `CORE_WAITING_SME_ENGINE_CYCLE`, and six load/store uop counters. These are reachable via the standard kperf/kpc interface. M5 replaces Apple's opaque AMX with ARM-standard SME, and Apple has wired up dedicated engine-level performance counters. No equivalent events appear in M1–M4 kpep dumps.
- **Why it matters:** Directly resolves the "AMX event discovery" open problem for M5+: `CORE_WAITING_SME_ENGINE_CYCLE` (stall cycles waiting for the SME engine) and `INST_SME_ENGINE_ALU` (retired ALU instructions) are now measurable through the existing kperf sidecar, enabling hardware-counter-based matrix-coprocessor utilization on M5 hardware.

### Finding 2: kennss/SiliconScope — new ANE-bandwidth monitor with IOReport channel debug mode

- **Source:** kennss/SiliconScope (GitHub)
- **URL:** https://github.com/kennss/SiliconScope
- **Date:** Released June 20–24, 2026
- **Summary:** SiliconScope is a new native SwiftUI macOS monitor (no sudo) that explicitly exposes ANE bandwidth and Media Engine metrics via IOReport. Its CLI companion (`sscope-cli --power-debug`) dumps every IOReport power channel across all groups on the running hardware, providing a practical tool for mapping which channel names carry ANE power on new chip generations. A curated channel map is maintained at `docs/ioreport-channels.md`.
- **Why it matters:** The `--power-debug` dump accelerates mapping M5 ANE IOReport channel names — which t3rm1nu55-monitorplus needs before it can expose ANE power on M5. Adds a new untracked tool to the reference set.

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
