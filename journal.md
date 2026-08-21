# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-21 — sweep (5 findings)

### Finding 1: maderix/ANE — Transformer training on ANE via reverse-engineered private APIs
- **Source:** maderix (tracked Substack author — new GitHub repo)
- **URL:** https://github.com/maderix/ANE
- **Date:** March 3–10, 2026 (42 commits, 7.2k stars)
- **Summary:** maderix published a GitHub repo implementing forward+backward transformer training on ANE via `_ANEClient`/`_ANECompiler`, with INT8 W8A8 quantization (1.88× throughput) and GPU↔ANE zero-copy via shared IOSurface memory. ANE utilization is estimated at 5–9% of peak from throughput benchmarks only — no hardware performance counters are discovered or exposed.
- **Why it matters:** Confirms the `_ANEClient` private API call sequence is stable enough for a 42-commit production repo; the gap between 5–9% utilization estimate (benchmark-derived) and 100% reinforces that hardware counter access remains the unsolved piece.

### Finding 2: arXiv 2603.06728 — Orion: First complete LLM training on ANE, 14 new documented constraints
- **Source:** arXiv (new paper, not previously tracked)
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026
- **Summary:** Orion (Ramchand Kumaresan) is the first open end-to-end system for LLM training+inference directly on ANE, bypassing CoreML via `_ANEClient`/`_ANECompiler`. Catalogs 20 ANE constraints on MIL IR programs (memory layout, compilation limits, numerical behavior) — 14 of which were previously undocumented. Companion GitHub: mechramc/Orion.
- **Why it matters:** Extends the public knowledge of ANE programming-model constraints; 14 newly documented constraints are directly relevant if t3rm1nu55-monitorplus ever attempts to dispatch a synthetic ANE workload for utilization probing.

### Finding 3: arXiv 2606.22283 — Definitive ANE architecture reverse-engineering paper + guide
- **Source:** arXiv (new paper, not previously tracked)
- **URL:** https://arxiv.org/abs/2606.22283 — guide: https://github.com/sbryngelson/ane-guide
- **Date:** June 21, 2026
- **Summary:** Spencer H. Bryngelson (Georgia Tech) published a comprehensive reverse-engineered account of ANE based on direct measurement and static analysis of the private runtime, compiler, kernel driver, and firmware. Documents the datapath and roofline bounds, the dispatch route below CoreML, compiler and on-disk program format, weight-compression scheme, and the kernel driver/firmware/command protocol. Companion reference guide at sbryngelson/ane-guide.
- **Why it matters:** Now the definitive public ANE reference. The dispatch-route and driver-protocol documentation are directly relevant if t3rm1nu55-monitorplus eventually attempts ANE utilization probing via driver-level hooks or IOReport cross-correlation.

### Finding 4: arXiv 2606.17090 — ANEForge: Python direct ANE computation without CoreML
- **Source:** arXiv (new paper, not previously tracked)
- **URL:** https://arxiv.org/abs/2606.17090 — package: https://pypi.org/project/aneforge/ — GitHub: https://github.com/sbryngelson/ANEForge
- **Date:** June 12, 2026
- **Summary:** ANEForge (Spencer H. Bryngelson) is a Python package that compiles a lazy tensor graph of 58 fused operators and 19 native bridge operators into a single ANE program and dispatches it through the ANE daemon and kernel-driver stack, fully bypassing CoreML. Available on PyPI.
- **Why it matters:** Demonstrates a reproducible operator-to-ANE-program pipeline usable as a reference for constructing synthetic ANE workloads to probe activity indirectly (e.g., detecting ANE-active state from IOReport energy deltas).

### Finding 5: darwin-kperf Rust crate — kperf/kperfdata/kpc FFI for Rust
- **Source:** crates.io (new, not previously tracked)
- **URL:** https://crates.io/crates/darwin-kperf — docs: https://docs.rs/darwin-kperf
- **Date:** Discovered August 2026
- **Summary:** darwin-kperf is a Rust crate wrapping Apple's private kperf.framework and kperfdata.framework via dlopen (no link-time dependency on private headers), exposing the full kperf/kperfdata/kpc API surface for reading hardware PMU counters (cycles, instructions, cache events) on M1–M5. Companion sys crate darwin-kperf-sys; criterion integration via darwin-kperf-criterion. Requires root or `com.apple.private.kernel.kpc` entitlement.
- **Why it matters:** t3rm1nu55-monitorplus is written in Rust and needs exactly this kperf FFI for its privileged PMU sidecar. darwin-kperf may be directly usable instead of a hand-rolled FFI — warrants a direct evaluation.

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
