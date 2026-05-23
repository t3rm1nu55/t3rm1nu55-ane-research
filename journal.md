# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-23 — sweep (3 findings)

Two findings are from the seed period (pre-2026-04-07) that the manual synthesis missed; one is new since the last check.

### Finding 1: Orion — open end-to-end ANE system via `_ANEClient`, confirms 119-compile limit
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** 2026-03-06 (submitted); missed in initial seed
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" bypasses CoreML entirely by driving `_ANEClient` and `_ANECompiler` private APIs directly, building a compiler pipeline that supports forward and backward passes. It confirms 19 TFLOPS FP16 on M4 at 2.8W, measures 94% ANE utilization only when chaining 32+ ops into a single compiled graph, and discovers a hard ~119-compilation-per-process limit enforced by the ANE kernel extension.
- **Why it matters:** The 119-compile cap is a hard constraint for any ANE telemetry probe that dispatches synthetic workloads; the utilization curve (graph depth vs. throughput) is now the published SOTA methodology for inferring ANE activity without hardware counters.

### Finding 2: `darwin-kperf` — Rust crate wrapping Apple kperf for M1–M5 hardware counters
- **Source:** crates.io
- **URL:** https://crates.io/crates/darwin-kperf
- **Date:** 2026-02-23 (v0.1.0 released); missed in initial seed
- **Summary:** A new Rust crate (v0.1.0, released Feb 23 2026) that wraps Apple's private `kperf.framework` and `kperfdata.framework` to expose hardware performance counters on Apple Silicon (M1–M5). It provides the same counter infrastructure as Instruments/xctrace with low overhead. No AMX- or ANE-specific events are exposed; coverage is standard CPU PMU events (cycles, instructions, cache miss, branch). No ABI stability guarantee.
- **Why it matters:** t3rm1nu55-monitorplus uses a privileged sidecar for kperf access with a custom Rust FFI; darwin-kperf is a ready-made binding that could replace or cross-validate that implementation, reducing maintenance surface.

### Finding 3: AsahiLinux/m1n1 — M5 and A18 Pro (MacBook Neo) hardware bring-up started
- **Source:** AsahiLinux/m1n1 GitHub
- **URL:** https://github.com/AsahiLinux/m1n1/commits/main/
- **Date:** 2026-05-06 to 2026-05-15
- **Summary:** Two May 2026 m1n1 commits mark initial hardware bring-up for new Apple SoCs: "Initial support for T8140" (May 6) covers what is believed to be the M5 SoC, and "pmgr: support M4 Pro/Max / A18 Pro / M5" (May 15) adds power-management register tables for M5 and the A18 Pro (the chip in Apple's MacBook Neo, a new Mac shipping a mobile SoC). This is early-stage bootloader work, not yet PMU counter documentation.
- **Why it matters:** Asahi's register-documentation pipeline for new chips follows bringup with a 6–12 month lag; M5 and A18 Pro PMU event tables are the logical next milestone, and the A18 Pro in a macOS device is new monitoring territory that may have different IOReport channel names than M-series.

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
