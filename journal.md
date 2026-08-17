# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-17 — sweep (6 findings)

### Finding 1: Comprehensive ANE reverse-engineering paper and guide (arxiv 2606.22283)
- **Source:** arXiv / Spencer H. Bryngelson (Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.22283 — web edition: https://ane-guide.readthedocs.io
- **Date:** June 21, 2026
- **Summary:** "Apple Neural Engine: Architecture, Programming, and Performance" is the most complete public reverse-engineering of the ANE to date, covering the datapath and roofline, the dispatch path below CoreML via `_ANEClient`/`_ANECompiler`, the on-disk program format and weight-compression scheme, and the kernel driver, firmware, and command protocol. Based on direct hardware measurement and static analysis of the private runtime.
- **Why it matters:** This is now the reference document for ANE internal structure; any future ANE counter-access work in t3rm1nu55-monitorplus should start here.

### Finding 2: ANEForge — PyPI-installable Python package for direct ANE computation (arxiv 2606.17090)
- **Source:** arXiv / Spencer H. Bryngelson; GitHub: https://github.com/sbryngelson/ANEForge
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 12, 2026
- **Summary:** ANEForge compiles a lazy tensor graph (58 fused operators, 19 native bridge operators) into a single ANE program, bypassing CoreML entirely. Available on PyPI as `aneforge`. Companion to the 2606.22283 paper.
- **Why it matters:** First installable Python library for direct ANE access — lowers the bar for experimenting with ANE dispatch patterns and measuring throughput outside CoreML.

### Finding 3: Orion — first open end-to-end ANE training+inference system (arxiv 2603.06728)
- **Source:** arXiv / Ramchand Kumaresan; GitHub: https://github.com/mechramc/Orion
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026
- **Summary:** Orion bypasses CoreML via `_ANEClient`/`_ANECompiler` and is the first publicly available system that combines direct ANE execution, a compiler pipeline, and stable multi-step training with checkpoint resume. The paper catalogs 20 previously undocumented restrictions on MIL IR programs, memory layout, and numerical behavior.
- **Why it matters:** The constraint catalog is directly useful for understanding why certain IOReport power signatures appear; the GitHub repo is a live reference implementation for `_ANEClient` calling conventions.

### Finding 4: maderix/ANE — direct ANE backpropagation with utilization measurement
- **Source:** GitHub: https://github.com/maderix/ANE
- **Date:** Active in 2026 (companion to Part 3 of the maderix Substack series)
- **Summary:** Extends the earlier maderix reverse-engineering work to full backpropagation on ANE hardware (no CoreML, no Metal). Uses `_ANEClient`, `_ANECompiler`, and `_ANEInMemoryModelDescriptor`; exposes intermediate activations via forward taps; uses IOSurface for zero-copy tensor I/O. Reports 11.2% ANE utilization at 1.78 TFLOPS sustained for a single transformer layer — the first public utilization figure derived from direct ANE dispatch.
- **Why it matters:** The utilization figure (11.2%) is derived from wall-clock timing against a known FP16 ceiling — this is the closest existing proxy for "ANE utilization %" and shows the methodology we could replicate in t3rm1nu55-monitorplus using IOReport energy deltas paired with known benchmark throughput.

### Finding 5: "Above the Inner Loop" — first microbenchmark of M1 AMX inner loop (arxiv 2606.25426)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Finds the M1 AMX inner loop is load-issue bound, falling to 610–680 GFLOPS under any operand-load interleaving (vs. a load-free rate roughly 2× higher). Reveals the M1 has two on-chip AMX blocks (one per P-core) and achieves 1.17× speedup over Accelerate's fastest FP32 GEMM path by exploiting fine multi-thread paneling across both blocks. First paper to directly characterize AMX throughput limits rather than measuring end-to-end algorithmic performance.
- **Why it matters:** Establishes the AMX roofline for M1; confirms that AMX throughput is not visible via kperf (no AMX-specific events are used) — power/energy indirection remains the only viable proxy.

### Finding 6: jiegec/apple-pmu — kpep dumps for M4 (as4) and M5 (as5) including SME engine counters
- **Source:** GitHub: https://github.com/jiegec/apple-pmu
- **Date:** Ongoing; as4.md and as5.md present as of August 2026
- **Summary:** This untracked repo dumps the full `/usr/share/kpep` event database across all Apple Silicon generations. The as4 (M4) database adds ARM architectural events and SME engine counters; as5 (M5) adds load data source tracking (LD_SRC_*) and PL2 cache events. The presence of SME engine counters in the M4 kpep database is new — ARM SME is the documented replacement for Apple's proprietary AMX encoding on M4+.
- **Why it matters:** If SME counters are programmable via kperf on M4+, they could serve as the first hardware-counter-based AMX proxy on those chips. This warrants direct investigation in the kperf sidecar.

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
