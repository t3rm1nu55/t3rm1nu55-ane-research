# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-19 — sweep (7 findings)

### Finding 1: Full ANE Architecture/Programming paper — arXiv 2606.22283
- **Source:** arXiv (Spencer H. Bryngelson, Georgia Institute of Technology)
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 2026
- **Summary:** The most comprehensive reverse-engineered account of the ANE published to date. Documents the full ANE datapath, roofline throughput/energy bounds, the dispatch route below CoreML, compiler and on-disk program format, weight compression, kernel driver, firmware, and command protocol — all via direct measurement on Apple silicon and static analysis of the private runtime/compiler/firmware stack. The paper explicitly maps the IOKit command ring and driver ABI Apple uses internally.
- **Why it matters:** Driver-level documentation is the prerequisite to finding any utilization counter surface; this paper may identify the register or mailbox interface where throughput metrics live inside the ANE daemon.

### Finding 2: ANEForge — Python package for direct ANE dispatch
- **Source:** arXiv (Spencer H. Bryngelson, Georgia Institute of Technology)
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 12, 2026
- **Summary:** Companion to Finding 1. A Python package that compiles a lazy tensor graph (58 fused operators, 19 native bridge operators) directly to an ANE program, dispatched through the same ANE daemon and kernel driver Apple uses internally — entirely bypassing CoreML. Supports both inference and training (forward/backward pass, Adam optimizer with 109M parameters).
- **Why it matters:** ANEForge is the first MIT-licensed tool that drives the ANE at the driver ABI level; its source is a direct reference for any attempt to intercept or instrument ANE dispatch for utilization metrics.

### Finding 3: AMX GEMM paper — Exceeding Accelerate on M1 AMX
- **Source:** arXiv (Deyvik Bhan, Georgia Institute of Technology)
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Reverse-engineers the Apple Matrix Extension (AMX) on M1 through M3, characterizes GEMM dispatch to AMX during LLM prefill, and demonstrates optimizations that beat Apple's own Accelerate framework. Directly characterizes AMX throughput at the microarchitectural level via black-box benchmarking.
- **Why it matters:** Provides the most detailed public AMX performance characterization since the MIT CSAIL thesis; if the paper discloses any kperf/PMU event used during measurement, that would be a new AMX event lead.

### Finding 4: m1n1 — Initial M5 (T6050/T6051) support with new "M-Core" type
- **Source:** AsahiLinux/m1n1 (commit b2d3f5e, 2026-07-10)
- **URL:** https://github.com/AsahiLinux/m1n1/commit/b2d3f5eba7ced3f5f9e00044744d2c4b4f2efeba
- **Date:** 2026-07-10 (committed; authored 2026-06-29)
- **Summary:** Asahi Linux's m1n1 gains preliminary M5 Pro boot support (chip IDs T6050/T6051). A significant architectural detail: the M5 introduces a third core type, "M-Core" (cluster-type `m` in ADT, MPIDR `0x80040X0Y`), distinct from P-Core and E-Core. M-Core lacks the `SYS_IMP_APL_E_*` sysregs present on E-Cores. A companion commit (5d84c4c) adds MPIDR-based M-Core detection to prevent misidentification.
- **Why it matters:** A new core type means potentially new or renamed PMU event IDs; kperf counter enumeration for M5 cannot assume M4-compatible event lists, and any future `dougallj/applecpu` M5 documentation will need to cover M-Core events.

### Finding 5: m1n1 — AMX register access gated by apple_sysregs_unlocked on newer chips
- **Source:** AsahiLinux/m1n1 (commits 001637f and 53d4271, 2026-07-13)
- **URL:** https://github.com/AsahiLinux/m1n1/commit/001637fab15f712d84dbad87c8e7d32d4d7594fd
- **Date:** 2026-07-13
- **Summary:** Two commits gate writes to the AMX (and SPRR/GXF) system registers behind a new `apple_sysregs_unlocked` flag, which is not set on newer chips such as A18 Pro / M4 Pro. Previously these registers were written unconditionally; on newer silicon, attempting to write them without the unlock causes a fault.
- **Why it matters:** Any attempt to read AMX-adjacent sysregs for counter values on M4 Pro or M5 will require the same unlock sequence; this documents the specific hardware gate that must be cleared before AMX sysreg access is possible.

### Finding 6: macmon — Active residency ratios exposed via IOReport
- **Source:** vladkens/macmon (commit 3010f1f, 2026-06-09)
- **URL:** https://github.com/vladkens/macmon/commit/3010f1fb7e209c334a1f948ea4386d92e1d761d2
- **Date:** 2026-06-09
- **Summary:** macmon v0.8 (unreleased tag) adds IOReport active residency ratio metrics, referencing issue #61. This exposes the per-cluster CPU active residency fraction — a metric derived from existing IOReport channels that was previously unexplored in the Rust IOReport codebase.
- **Why it matters:** Active residency ratios from IOReport are the closest proxy available for per-cluster "utilization" without kperf; this implementation is a direct pattern to replicate in t3rm1nu55-monitorplus for E-cluster vs P-cluster active-time fractions.

### Finding 7: Residual GPU Cache paper — kperf/PMU methodology on Apple M4 Pro
- **Source:** arXiv (Faruk Alpay, Barış Başaran, Bahçeşehir University)
- **URL:** https://arxiv.org/abs/2606.27098
- **Date:** June 25, 2026
- **Summary:** Characterizes GPU-to-CPU cache pollution on M4 Pro using PMU counters read via kperf/kpc (requiring root) and IOReport DCS-agent histograms. Uses L1D refill PMU counters to distinguish 64-byte cache sectors from 128-byte software-visible lines. Documents the root-only kperf counter access flow on M4 Pro silicon specifically.
- **Why it matters:** Confirms that the kperf/kpc root-access pattern documented in ibireme's gist continues to work on M4 Pro; the specific PMU counter IDs used in the paper are worth extracting as an M4 Pro counter dataset.

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
