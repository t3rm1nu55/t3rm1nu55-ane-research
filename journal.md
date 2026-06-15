# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-15 — sweep (4 findings)

### Finding 1: `darwin-kperf` Rust crate ecosystem — Criterion integration for Apple Silicon PMCs
- **Source:** crates.io / docs.rs
- **URL:** https://crates.io/crates/darwin-kperf
- **Date:** 2026-02-23 (initial release of darwin-kperf-criterion)
- **Summary:** Bilal Mahmoud (HASH) published a family of Rust crates wrapping Apple's private kperf.framework: `darwin_kperf_sys` (raw FFI), `darwin-kperf` (safe wrapper), and `darwin-kperf-criterion` (Criterion.rs `Measurement` implementation backed by hardware PMCs). The criterion integration lets Rust benchmark suites use PMC counts—cycles, instructions, cache misses—instead of wall-clock time, falling back to wall-clock on non-macOS. Root privileges still required for full counter access.
- **Why it matters:** An open-source Rust kperf FFI already exists and is actively maintained; the main project's privileged sidecar should evaluate this as a replacement or reference for its own kperf bindings before writing from scratch.

### Finding 2: Orion — first open end-to-end LLM system directly on the ANE (arXiv 2603.06728)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** 2026-03-06
- **Summary:** Ramchand Kumaresan's Orion is the first open system combining direct ANE execution, a compiler pipeline, and stable multi-step LLM training—bypassing CoreML entirely via `_ANEClient` and `_ANECompiler` private APIs. The paper catalogues 20 concrete restrictions on MIL IR programs that govern what the ANE will accept, and achieves GPT-2 124M at 170+ tok/s and Stories110M training in 22 minutes on M4 Max. No new hardware performance counters are exposed; utilization is derived from wall-clock timing.
- **Why it matters:** The 20 MIL IR restriction catalog is the most detailed public specification of ANE runtime behavior to date; confirms no counter API surface exists and that black-box timing remains the only public measurement primitive.

### Finding 3: maderix/ANE GitHub repo — benchmarking code with SRAM layout probing
- **Source:** GitHub
- **URL:** https://github.com/maderix/ANE
- **Date:** 2026-03-10 (most recent commit)
- **Summary:** Companion code to maderix's Substack ANE series (already tracked). Contains `inmem_peak.m` (peak TOPS measurement: 18.6 TOPS FP16, 35.1 TOPS INT8 on M4), `sram_bench.m` / `sram_probe.m` (ANE SRAM bandwidth and layout exploration), and `ane_int8_bench.m` (W8A8 quantization throughput). All measurement is via wall-clock timing and operational counts, not hardware counters.
- **Why it matters:** `sram_probe.m` explores the internal SRAM geometry of the ANE, which is the closest any public code has come to probing ANE internals; this is a reference for any future hardware-level ANE characterization work.

### Finding 4: `mperf` — practical kperf CLI with portable event aliases and kpep database paths
- **Source:** lambdafoo.com (Perpetually Curious Blog)
- **URL:** https://lambdafoo.com/posts/2026-03-25-mperf-hardware-counters-macos.html
- **Date:** 2026-03-25
- **Summary:** Introduces `mperf`, a CLI and library wrapping kperf/kperfdata that provides portable event-name aliases (cycles, instructions, branch-misses, l1d-cache-misses, etc.) resolving to the correct chip-specific identifiers. Documents the per-chip kpep database paths under `/usr/share/kpep/`: `a14.plist` for M1, `a15.plist` for M2, `as4.plist` for M4. Apple Silicon provides 2 fixed counters plus 8 configurable counters (10 total simultaneous events).
- **Why it matters:** Confirms and documents the kpep database path naming scheme for M4 (`as4.plist`); the portable alias approach is a model for the main project's counter selection API.

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
