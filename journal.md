# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-04-15 — sweep (3 findings)

### Finding 1: M4 PMU regression — `kpc_set_config` fails for configurable counters; M3/M4 PMESR encoding changed
- **Source:** CLF3's blog
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** Unknown (not date-indexed; surfaced during sweep)
- **Summary:** Documents that on M4, `kpc_set_config` silently fails when any configurable counter slot is specified; only the two fixed counters (cycles, instructions) continue to work. Also documents that M3/M4 changed PMESR event encoding from 8-bit per slot (M1/M2) to 16-bit per slot, and that the `SYS_APL_PMCR0_EL1` register is overwritten by a kernel thread roughly every 100 µs on all generations, requiring a patched kernel for sustained counter access.
- **Why it matters:** The monitorplus kperf sidecar calls `kpc_set_config` to program configurable counter events; this call fails silently on M4 hardware, meaning all PMU event data beyond cycles/instructions is currently zero or garbage on M4 — requires an M4-specific code path and updated PMESR event encoding.

### Finding 2: Orion — first open direct-ANE runtime, bypasses CoreML, documents 20 constraints
- **Source:** arXiv + GitHub
- **URL:** https://arxiv.org/abs/2603.06728 · https://github.com/mechramc/Orion
- **Date:** March 6, 2026
- **Summary:** Orion is the first open end-to-end system for ANE programming that bypasses CoreML via `_ANEClient` and `_ANECompiler` private APIs, and demonstrates both inference and backward-pass training on ANE. It catalogs 20 MIL IR constraints (14 previously undocumented), including a 32 MB SRAM performance cliff and the finding that INT8 is dequantized to FP16 before compute. ANE utilization is measured via wall-clock ratio of `orion_eval` to total wall time — no hardware performance counter access.
- **Why it matters:** The constraint catalog and `_ANEClient`/`_ANECompiler` depth advance the field's understanding of the ANE execution model; the wall-clock utilization approach confirms no counter-based ANE utilization metric has been found, keeping that as the main open problem. The `_ANECompiler` symbol surface may be a hook point for future monitoring work.

### Finding 3: k06a/macpow — untracked IOReport tool with fabric interconnect channel coverage
- **Source:** GitHub
- **URL:** https://github.com/k06a/macpow
- **Date:** Unknown
- **Summary:** macpow is a no-sudo power-tree TUI that reads IOReport Energy Model and surfaces fabric interconnect sub-components (AMCC, DCS, FAB, AFR channels) not exposed by macmon or socpowerbud. Supports M1–M5 including multi-die Ultra configurations with generic channel-name parsing for forward compatibility.
- **Why it matters:** The AMCC (Apple Memory Controller Complex) power channel may carry indirect signatures useful for AMX activity inference; the broader channel coverage is worth tracking alongside macmon as a reference for undocumented IOReport channel names.

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
