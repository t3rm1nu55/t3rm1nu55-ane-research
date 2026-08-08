# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-08 — sweep (6 findings)

### Finding 1: "Apple Neural Engine: Architecture, Programming, and Performance" (arXiv 2606.22283)
- **Source:** arXiv / ane-guide.readthedocs.io
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 2026
- **Summary:** Spencer H. Bryngelson's comprehensive reverse-engineered account of the ANE based on direct measurement of Apple Silicon and static analysis of the private runtime, compiler, kernel driver, and firmware. Documents the datapath and roofline bounding throughput and energy, the dispatch route below CoreML, the on-disk program/weight-compression format, and crucially the kernel driver, firmware, and command protocol. A companion web guide is hosted at https://ane-guide.readthedocs.io and source at https://github.com/sbryngelson/ane-guide.
- **Why it matters:** The most complete public documentation of the ANE hardware stack ever published. The kernel driver and command-protocol section is the closest anyone has publicly come to describing what registers the host CPU uses to communicate with the ANE — foundational for any future ANE hardware counter work in t3rm1nu55-monitorplus.

### Finding 2: maderix/ANE — first open training on ANE with utilization proxy metric
- **Source:** GitHub maderix/ANE
- **URL:** https://github.com/maderix/ANE
- **Date:** March–May 2026 (ongoing; original Substack blog already tracked)
- **Summary:** First open-source demonstration of forward+backward-pass neural network training directly on the ANE via reverse-engineered private APIs (`_ANEClient`, `_ANECompiler`, `_ANEInMemoryModelDescriptor`). Reports ANE utilization as a power-normalized percentage (5–9% of theoretical peak for current workloads; 11.2% for a specific transformer layer at 1.78 TFLOPS vs 15.8 TFLOPS peak). Also documents that Apple's "38 TOPS" M4 ANE claim appears to be a marketing convention of doubling the FP16 figure.
- **Why it matters:** Establishes the first public power-derived ANE utilization proxy metric, and confirms `_ANEInMemoryModelDescriptor` as a new in-memory compilation pathway below CoreML. The utilization approach validates the power-indirection strategy already used in t3rm1nu55-monitorplus.

### Finding 3: darwin-kperf — production Rust crate for kperf/kperfdata FFI
- **Source:** crates.io / GitHub (author: Bilal Mahmoud / HASH)
- **URL:** https://crates.io/crates/darwin-kperf
- **Date:** February 23, 2026 (not tracked in prior sweep)
- **Summary:** A Rust crate wrapping Apple's private `kperf.framework` and `kperfdata.framework` via `dlopen` with no link-time dependency on private headers. Loads event databases from `/usr/share/kpep/*.plist` at runtime, supports M1–M5. Requires root or `com.apple.private.kernel.kpc` entitlement. A companion `darwin-kperf-criterion` crate integrates with Criterion benchmarks.
- **Why it matters:** This is exactly the kperf FFI that t3rm1nu55-monitorplus's privileged sidecar needs. It is already published, MIT/Apache-2.0 licensed, and Rust-native — a direct adoption candidate that should be evaluated before writing a bespoke FFI.

### Finding 4: arXiv 2606.25426 — AMX microarchitecture: load-issue bound characterization
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 24, 2026
- **Summary:** Deyvik Bhan shows via microbenchmarks that M1 AMX throughput is load-issue bound — any operand load interleaving with the FMA32 stream drops single-thread throughput from the load-free ceiling to ~610–680 GFLOPS. A direct-AMX GEMM kernel using fine multi-thread panels (filling M1's second on-chip AMX block for K≥N shapes) and pre-packed weights achieves a 1.58× geometric mean speedup over `BNNSMatMul` across 12 LLM prefill shapes.
- **Why it matters:** The first public microbenchmark characterization of AMX memory-access constraints. Confirms AMX throughput is bounded by the load path, not compute — which shapes what "AMX utilization" would mean as a metric and suggests load-bandwidth counters (L2 load requests) are the best indirect proxy.

### Finding 5: M5 IOReport channel rename — PMP → PMP0 in bandwidth group
- **Source:** vladkens/macmon releases + kennss/SiliconScope
- **URL:** https://github.com/vladkens/macmon / https://github.com/kennss/SiliconScope
- **Date:** 2026 (M5 Max launch window)
- **Summary:** Both macmon and SiliconScope independently discovered and patched breaking changes in M5 IOReport channels: the bandwidth group key `PMP` was renamed `PMP0` on M5 hardware, and the voltage-states keys were renumbered on M5 Max, causing crashes. macmon also fixed E/P/S core label parsing for M5's new topology. SiliconScope documents that ANE "usage" remains a power-normalized estimate — Apple still doesn't expose ANE occupancy.
- **Why it matters:** t3rm1nu55-monitorplus must handle the `PMP → PMP0` rename for M5 compatibility. SiliconScope's per-generation channel table approach is the reference pattern for implementing this.

### Finding 6: CLF3 blog — M3/M4 PMU register architecture differs from M1/M2
- **Source:** ClF3's blog
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** 2026 (exact date unconfirmed)
- **Summary:** Documents that M3/M4 PMU ESR registers are 64-bit with each event encoded in 16 bits (vs different field layout on M1/M2), and that performance counters themselves are 64 bits with bit 63 triggering PMI. Provides concrete register definitions for `SYS_APL_PMCR0_EL1` and `SYS_APL_PMCR1_EL1` on M3/M4.
- **Why it matters:** The kperf privileged sidecar will encounter these per-generation differences in raw register layout if it ever drops below the kpep abstraction layer. The `/usr/share/kpep/*.plist` approach hides this, but direct register access requires knowing it.

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
