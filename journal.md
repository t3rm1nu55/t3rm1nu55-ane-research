# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-17 — sweep (3 findings)

### Finding 1: Comprehensive ANE reverse-engineering guide (A11–M5) published on arXiv

- **Source:** sbryngelson/ane-guide + arXiv 2606.22283
- **URL:** <https://arxiv.org/abs/2606.22283> / <https://github.com/sbryngelson/ane-guide>
- **Date:** June 23, 2026
- **Summary:** Steven Bryngelson published a fully reverse-engineered reference for the Apple Neural Engine covering A11 through A18 and M1 through M5. It documents the datapath/roofline, below-CoreML dispatch route, compiler and on-disk program format, weight-compression scheme, and the kernel driver, firmware, and command protocol — derived from direct measurement on M1 and M5 plus static analysis of private binaries. Per-chip target tables and an operation-by-device matrix are included.
- **Why it matters:** The kernel driver and command protocol documentation is the layer where any ANE counter or utilization hook would need to land; this is now the deepest public reference available and supersedes the hollance/neural-engine notes for M4+ chips.

### Finding 2: ANEForge — direct ANE dispatch from Python, bypassing CoreML

- **Source:** sbryngelson/ANEForge + arXiv 2606.17090
- **URL:** <https://arxiv.org/abs/2606.17090> / <https://github.com/sbryngelson/ANEForge>
- **Date:** June 12, 2026
- **Summary:** From the same author as Finding 1, ANEForge is a Python library that compiles a lazy tensor graph (58 fused operators, 19 bridge operators) into a native ANE program and dispatches it through the ANE daemon and kernel-driver stack without touching CoreML. It supports full forward/backward passes and training, and achieves ~90 µs per dispatch call vs the engine's ~70 µs floor. Targets macOS 14+ on Apple Silicon.
- **Why it matters:** A working open-source direct-dispatch path to the ANE means the kernel driver handshake is now publicly documented and reproducible; any timing or energy measurement instrumented around the dispatch boundary would yield ANE utilization proxies without CoreML overhead.

### Finding 3: `darwin-kperf` — Rust crate wrapping Apple's private kperf/kpc framework

- **Source:** crates.io / darwin-kperf
- **URL:** <https://crates.io/crates/darwin-kperf>
- **Date:** First published February 23, 2026 (not captured in initial seed)
- **Summary:** A safe Rust wrapper around Apple's private `kperf.framework` and `kperfdata.framework` providing bindings for KPC (counter class configuration, register programming, per-thread counter reads) and KPEP (PMC event database from `/usr/share/kpep/`). A companion `darwin-kperf-criterion` crate exposes hardware PMC-backed criterion.rs measurements. Requires root or `com.apple.private.kernel.kpc` entitlement.
- **Why it matters:** This is exactly the Rust FFI layer that t3rm1nu55-monitorplus's kperf sidecar needs. It should be evaluated as a direct dependency or as the canonical reference implementation for the sidecar's counter-programming code — avoiding reimplementation of the struct layout reverse engineering.

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
