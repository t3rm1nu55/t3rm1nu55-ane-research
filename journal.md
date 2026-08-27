# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-27 — sweep (6 findings)

### Finding 1: ANE Architecture — Comprehensive Reverse Engineering
- **Source:** arXiv (Spencer H. Bryngelson, Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 21, 2026
- **Summary:** First comprehensive reverse-engineered characterization of the ANE covering A11 through M5: datapath, roofline throughput and energy bounds, the full dispatch route below CoreML, the on-disk HWX program format, weight compression scheme, and the kernel driver → firmware command protocol. Based on static analysis of the private runtime, compiler, kernel driver, and firmware; an extended online guide accompanies it at ane-guide.readthedocs.io. This is architecture documentation, not a new hardware counter.
- **Why it matters:** The kernel driver → firmware command protocol is the closest public knowledge yet to what direct ANE telemetry would look like; essential reference once ANE work begins post-v1.

### Finding 2: ANEForge — Direct ANE Dispatch Bypassing CoreML
- **Source:** arXiv / GitHub (Spencer H. Bryngelson, Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.17090 · https://github.com/sbryngelson/ANEForge
- **Date:** June 12, 2026
- **Summary:** Python package that compiles lazy tensor graphs into fused e5rt programs dispatched directly through the ANE daemon, bypassing CoreML. Maps 58 fused ANE operators and 19 native bridge operators through the private daemon and kernel-driver stack; covers A11–M5 and is actively developed (318+ commits since April 2026).
- **Why it matters:** The working dispatch code and operator catalog are the most complete public reference for how to interface with ANE activity without CoreML; methodology is a template for future instrumentation.

### Finding 3: M3/M4 PMU ESR Register Width Change
- **Source:** blog.clf3.org
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** Post-April 2026 (exact date unconfirmed)
- **Summary:** On M3 and M4, PMU event-selection registers (ESR) are 64-bit wide with 16-bit per-event slots, versus 32-bit ESR with 8-bit slots on M1/M2. Performance counter registers are 64-bit on M2/M3/M4 with bit 63 as the PMI trigger. `SYS_APL_PMCR0_EL1` is periodically overwritten by a macOS kernel process, limiting userspace modification windows to ~100 µs without a kernel patch.
- **Why it matters:** The kperf sidecar must handle per-generation differences in ESR encoding; code that hardcodes 8-bit event slot positions will silently misconfigure on M3/M4, causing wrong counter readings.

### Finding 4: M1 AMX Load-Issue Bottleneck and Second Undocumented AMX Block
- **Source:** arXiv (Deyvik Bhan)
- **URL:** https://arxiv.org/pdf/2606.25426
- **Date:** June 2026
- **Summary:** Micro-benchmarks establish that the M1 AMX inner loop is load-issue bound: when any operand load interleaves with the FMA32 stream, single-thread throughput collapses to 610–680 GFLOPS (under half the load-free ceiling). The paper also identifies a second, previously undocumented on-chip AMX block accessible via fine-grained multi-thread panel scheduling, achieving 1.17× over Apple Accelerate at FP32 GEMM prefill.
- **Why it matters:** First concrete AMX performance model; the second undocumented AMX block is a new hardware surface that has not appeared in any prior public analysis.

### Finding 5: kperf/kpc Confirmed Functional on M4 Pro
- **Source:** arXiv (Faruk Alpay, Barış Başaran)
- **URL:** https://arxiv.org/abs/2606.27098
- **Date:** June 25, 2026
- **Summary:** Programs L1D load-miss, L1D refill, L2-TLB data-miss, and data-table-walk counters via kperf/kpc on M4 Pro, finding that Metal GPU command completion provides execution ordering but not CPU cache residency isolation. The counter configuration recipe is described in the paper.
- **Why it matters:** Confirms kperf is accessible and stable on M4 Pro; provides tested counter event codes for cache-related events on the M4 generation.

### Finding 6: macmon M3 Ultra IOReport DIE_N_ Channel Naming
- **Source:** vladkens/macmon v0.8.2
- **URL:** https://github.com/vladkens/macmon/releases/tag/v0.8.2
- **Date:** August 4, 2026
- **Summary:** Bug fix for M3 Ultra reveals that dual-die Ultra chips prefix per-core IOReport channel names with `DIE_0_` and `DIE_1_` (e.g., `DIE_0_ECPU_CPU0`, `DIE_1_PCPU_CPU0`). The full channel name string is the correct collision-free key; (cluster, core) tuple alone collides across dies.
- **Why it matters:** IOReport channel-name parsers must handle the `DIE_N_` prefix to correctly enumerate per-core metrics on M3/M4 Ultra systems.

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
