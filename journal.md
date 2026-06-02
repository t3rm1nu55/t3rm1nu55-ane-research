# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-02 — sweep (4 findings)

### Finding 1: darwin-kperf — Safe Rust bindings to Apple's kperf/kperfdata frameworks
- **Source:** crates.io (`darwin-kperf`, `darwin-kperf-sys`, `darwin-kperf-criterion`)
- **URL:** https://crates.io/crates/darwin-kperf
- **Date:** Last updated February 23, 2026
- **Summary:** Three-crate Rust ecosystem wrapping Apple's private kperf and kperfdata frameworks: `darwin-kperf-sys` provides low-level `#[no_std]` FFI bindings, `darwin-kperf` adds a safe API layer, and `darwin-kperf-criterion` integrates hardware PMU counters as a Criterion.rs measurement backend. Supports M1–M5; requires root or the `com.apple.private.kernel.kpc` entitlement.
- **Why it matters:** t3rm1nu55-monitorplus is written in Rust; this crate ecosystem could serve as the foundation for the privileged kperf sidecar instead of hand-rolled FFI, substantially reducing implementation risk and maintenance surface.

### Finding 2: clf3.org — PMU ESR register width differs on M3/M4 vs M1/M2
- **Source:** ClF3's blog
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** Unknown (surfaced in April–June 2026 search results)
- **Summary:** Documents that on M3 and M4, the PMU Event Selection Register (ESR) is 64-bit with each event occupying 16 bits, versus 8 bits per event slot on M1/M2. Code that configures kperf events using M1/M2 bit-packing will silently misconfigure the counter on M3/M4 without producing an obvious error.
- **Why it matters:** The kperf sidecar must detect chip generation before writing ESR values; failure to do so produces wrong counter readings on any M3/M4 host — a correctness bug, not a performance issue.

### Finding 3: mperf — Portable PMU event-alias CLI for Apple Silicon
- **Source:** lambdafoo.com ("Perpetually Curious" blog)
- **URL:** https://lambdafoo.com/posts/2026-03-25-mperf-hardware-counters-macos.html
- **Date:** March 25, 2026
- **Summary:** Introduces `mperf`, a `perf stat`-like CLI for macOS/Apple Silicon built on kperf/kperfdata. Its key design is a portable alias layer ("cycles", "instructions", "branch-misses", "l1d-cache-misses") that automatically resolves to the correct chip-specific event IDs across M1–M4. Author notes kperf/kperfdata have been ABI-stable across M1–M4 but carry no public guarantee.
- **Why it matters:** mperf's alias-resolution logic is a direct reference implementation for the generation-portability problem the kperf sidecar must solve; the chip-to-event-ID mapping table is worth extracting.

### Finding 4: maderix ANE series complete + maderix/ANE code repository
- **Source:** maderix Substack (Parts 1 and 3, previously only Part 2 was tracked); GitHub
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine — https://github.com/maderix/ANE
- **Date:** February 28 – March 7, 2026
- **Summary:** The seed captured Part 2 (benchmarks). Part 1 documents the full software stack from CoreML down to IOKit driver, cracks the binary format, and identifies a ~119-op compilation limit per process. Part 3 demonstrates transformer training on ANE via `_ANEClient`/`_ANECompiler`, scaling to Qwen3-0.6B. The companion code repo (`github.com/maderix/ANE`, 42 commits) achieves only 5–9% of peak ANE throughput during training.
- **Why it matters:** The 5–9% efficiency ceiling during training is direct evidence that the absence of real-time hardware counters is a genuine engineering gap; the code repo provides the most complete public reference for the `_ANEClient` private API surface.

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
