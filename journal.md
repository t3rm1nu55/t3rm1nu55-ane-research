# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-15 — sweep (2 findings)

### Finding 1: jiegec/apple-pmu — M5 kpep counter dump includes SME-specific events; no AMX events

- **Source:** jiegec/apple-pmu (newly discovered; not previously tracked)
- **URL:** https://github.com/jiegec/apple-pmu
- **Date:** as5.md initially committed January 8, 2026; refreshed from macOS 26.4 beta on March 25, 2026
- **Summary:** This repo extracts and renders all PMU counter definitions from macOS `/usr/share/kpep/` for every chip generation. The M5 file (`as5.md`) documents 169 counter events and includes **SME (ARM Scalable Matrix Extension) specific counters** — loads, stores, ALU ops, and mode transitions — with no equivalent AMX-specific entries. M1–M4 kpep files have no SME events; this appears in as5 only.
- **Why it matters:** If M5 uses ARM's standardized SME rather than Apple's proprietary AMX, observable SME PMU events may provide the first hardware-counter-derived matrix-op throughput metric on Apple Silicon, resolving the AMX counter open problem for M5+ hardware. Needs verification on M5 hardware.

### Finding 2: darwin-kperf — Rust crate providing kperf/kperfdata bindings, M1–M5

- **Source:** darwin-kperf crate (newly discovered; not previously tracked)
- **URL:** https://crates.io/crates/darwin-kperf
- **Date:** Published/updated February 23, 2026
- **Summary:** A Rust crate wrapping Apple's private `kperf.framework` and `kperfdata.framework`, exposing PMU counter access on Apple Silicon M1 through M5. Licensed MIT/Apache-2.0. Companion crate `darwin-kperf-criterion` adds Criterion.rs integration for counter-driven micro-benchmarking.
- **Why it matters:** monitorplus's kperf privileged sidecar is currently planned as hand-rolled FFI; adopting this crate reduces implementation risk, adds tested M5 support, and avoids re-doing the kperfdata struct-layout reverse-engineering work.

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
