# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-09-09 — sweep (6 findings)

### Finding 1: Comprehensive ANE reverse-engineering paper — architecture, dispatch path, kernel driver documented
- **Source:** arXiv / Spencer H. Bryngelson (Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.22283 · companion: https://ane-guide.readthedocs.io · https://github.com/sbryngelson/ane-guide
- **Date:** 2026-06-21
- **Summary:** "Apple Neural Engine: Architecture, Programming, and Performance" documents A11–M5 ANE via direct measurement on M1 and M5 plus static analysis of the private runtime, compiler, kernel driver, firmware, and command protocol. Covers the complete dispatch route below CoreML, the on-disk e5rt program format, and the weight-compression scheme — the most thorough public RE of the full ANE stack to date.
- **Why it matters:** The documented dispatch path and command protocol are prerequisites for any future native ANE utilization hook in t3rm1nu55-monitorplus.

### Finding 2: ANEForge — CoreML-free Python ANE dispatch (training included)
- **Source:** arXiv / Spencer H. Bryngelson
- **URL:** https://arxiv.org/abs/2606.17090 · https://github.com/sbryngelson/ANEForge · https://pypi.org/project/aneforge/
- **Date:** 2026-06-28
- **Summary:** ANEForge compiles lazy tensor graphs to single fused ANE programs and dispatches through the same daemon and kernel-driver stack Apple uses, without CoreML. Programs complete in ~90 µs (near the 70 µs dispatch floor). Training — forward pass, backward pass, Adam update — runs entirely on the ANE.
- **Why it matters:** First public proof of instrumented CoreML-free ANE dispatch; the dispatch surface is now documented well enough that a Rust FFI monitoring hook into the same path is plausible.

### Finding 3: ANE memory-controller byte counters used as utilization proxy
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2608.22110
- **Date:** 2026-08 (≈ 3 weeks before this sweep)
- **Summary:** "What actually runs: a measurement study of language model placement and decode speed on the Apple Neural Engine" uses the ANE's memory-controller byte counters during inference to determine whether compute actually landed on the engine (versus being silently routed to the CPU by CoreML). A 25.85M-parameter fp16 model showed zero bytes through the engine; the same graph in int8 showed ~83% ANE residency and ran 1.8–2.2× faster.
- **Why it matters:** **Critical finding** — demonstrates that the ANE has an externally-observable memory-controller byte counter that serves as a utilization proxy. This is the closest thing yet to a real-time ANE throughput metric and is a concrete research lead for t3rm1nu55-monitorplus.

### Finding 4: darwin-kperf — safe Rust kperf/kpc bindings for Apple Silicon
- **Source:** crates.io / hashdotai (Bilal Mahmoud)
- **URL:** https://crates.io/crates/darwin-kperf · https://docs.rs/darwin-kperf
- **Date:** first published 2026-02-23; actively maintained (v0.1.1 current)
- **Summary:** Rust crate ecosystem (darwin-kperf, darwin-kperf-sys, darwin-kperf-events, darwin-kperf-criterion) wrapping Apple's private kperf.framework and kperfdata.framework via dlopen at runtime. Provides safe access to all 10 programmable PMU counters on M1–M5 with event definitions included. Requires root or the `com.apple.private.kernel.kpc` entitlement.
- **Why it matters:** **Directly actionable** — t3rm1nu55-monitorplus is written in Rust. The darwin-kperf crate could replace a hand-rolled kperf FFI, with darwin-kperf-events providing the M1–M5 event catalog already maintained upstream.

### Finding 5: Orion — 14 previously undocumented ANE constraints catalogued
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** 2026-03-06 (published before last-checked date; missed by seed)
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" extends public knowledge of ANE MIL IR constraints to 20 total restrictions — 14 of which were previously undocumented — covering memory layout, compilation limits, numerical behavior, and op support. Notes that over two billion Apple devices ship with the ANE.
- **Why it matters:** Constraints on what the compiler accepts affect what workloads land on the engine; knowing them improves the accuracy of any workload-based ANE-active inference in t3rm1nu55-monitorplus.

### Finding 6: AMX→SME transition confirmed on M4+; SME has public Arm PMU event spec
- **Source:** arXiv / Deyvik Bhan (Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** 2026-06-24
- **Summary:** "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" explicitly confirms that M4 and later chips replaced Apple's proprietary AMX with Arm's Scalable Matrix Extension (SME). On M1, AMX is load-issue bound at ~610–680 GFLOPS single-thread; the M1 has two on-chip AMX blocks. The Arm SME specification includes a standard PMU event profile.
- **Why it matters:** M4+ AMX monitoring should target SME PMU events (defined in the Arm architecture spec and likely exposed through standard kperf channels) rather than trying to reverse-engineer Apple-proprietary AMX counters — a material change in research direction for t3rm1nu55-monitorplus's AMX track.

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
