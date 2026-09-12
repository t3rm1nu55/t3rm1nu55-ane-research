# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-09-12 — sweep (4 findings)

### Finding 1: "What actually runs" — ANE memory-controller byte counters confirmed
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2608.22110
- **Date:** August 2026
- **Summary:** Measurement study of LLM placement and decode speed on the ANE. Critically, it reads the ANE's **memory-controller byte counters** during inference to establish what actually executed — not just what the compiler intended. Finds placement is a property of how a computation is expressed (a fused RMSNorm is ANE-eligible; its arithmetically identical decomposition is CPU-only).
- **Why it matters:** Confirms that runtime byte-level counters on the ANE are accessible from user space, establishing a direct path to a real utilization signal — exactly the metric t3rm1nu55-monitorplus is missing.

### Finding 2: Bryngelson ANE Architecture Guide + ANEForge — comprehensive reverse engineering
- **Source:** arXiv / GitHub / readthedocs
- **URL:** https://arxiv.org/abs/2606.22283 (paper) · https://github.com/sbryngelson/ane-guide · https://arxiv.org/abs/2606.17090 (ANEForge) · https://github.com/sbryngelson/ANEForge
- **Date:** June 2026
- **Summary:** Spencer Bryngelson (Georgia Tech) published a reverse-engineered guide documenting the ANE's datapath and roofline, dispatch route below CoreML, compiler and on-disk program format, weight-compression scheme, and the kernel driver, firmware, and command protocol. Companion tool ANEForge provides Python bindings for direct ANE dispatch (bypassing CoreML) at ~90 µs latency, available on PyPI.
- **Why it matters:** The kernel driver and command protocol documentation is the most complete public treatment of the ANE's internal interface and is immediately useful for understanding what monitoring hooks might be accessible to a privileged process.

### Finding 3: Orion — first open ANE system with 20-constraint catalog
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026
- **Summary:** Orion bypasses CoreML entirely via Apple's private `_ANEClient` and `_ANECompiler` APIs to build the first open end-to-end system supporting direct ANE execution, a compiler pipeline, and on-device LLM training with checkpoint resume. Includes a catalog of 20 ANE constraints (14 previously undocumented) on MIL IR programs, memory layout, compilation limits, and numerical behavior.
- **Why it matters:** The private API call sequence (`_ANEClient` / `_ANECompiler`) is the exact interface a monitoring sidecar would use; the constraint catalog defines the programming model boundary.

### Finding 4: jiegec/apple-pmu — M5 kpep dump adds SME engine counters
- **Source:** GitHub
- **URL:** https://github.com/jiegec/apple-pmu
- **Date:** 2026 (active)
- **Summary:** This repo systematically extracts all PMU counter definitions from macOS `/usr/share/kpep` across A7–A19 and M1–M5. The M4-class chip ("as4") introduces ARM architectural events and `INST_SME_ENGINE_*` counters (ALU, LD, PACKING_FUSED, SCALARFP, ST); M5 ("as5") adds load data source tracking and PL2 cache events.
- **Why it matters:** The `INST_SME_ENGINE_*` events are the first kperf-accessible counters that may be AMX/SME-adjacent; if they're programmable via the existing kperf sidecar, this could be an incremental path to AMX utilization tracking on M4+ devices.

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
