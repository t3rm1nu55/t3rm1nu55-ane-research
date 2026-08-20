# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-20 — sweep (5 findings)

### Finding 1: "Apple Neural Engine: Architecture, Programming, and Performance" — Bryngelson (arXiv 2606.22283)
- **Source:** arXiv / sbryngelson/ane-guide (GitHub + readthedocs)
- **URL:** https://arxiv.org/abs/2606.22283 | https://github.com/sbryngelson/ane-guide
- **Date:** June 21, 2026
- **Summary:** Comprehensive reverse-engineered guide to ANE internals — datapath, roofline, kernel driver, firmware, command protocol, compiler, and weight-compression scheme — derived from direct measurement and static analysis of Apple's private runtime. This is the deepest public documentation of ANE internals ever published. Companion web edition at ane-guide.readthedocs.io.
- **Why it matters:** The kernel driver and command protocol documentation may reveal whether the ANE exposes a performance counter register — the core open question for t3rm1nu55-monitorplus.

### Finding 2: ANEForge — Python for direct ANE computation without CoreML (arXiv 2606.17090)
- **Source:** arXiv + PyPI + sbryngelson/ANEForge (GitHub)
- **URL:** https://arxiv.org/abs/2606.17090 | https://github.com/sbryngelson/ANEForge
- **Date:** June 12, 2026
- **Summary:** Python package compiling tensor graphs into ANE programs dispatched through the same private `aned`/kernel-driver stack CoreML uses internally, bypassing CoreML entirely. Supports 58 fused operators, 19 native bridge operators, and ANE-side training (forward, backward, Adam). Published on PyPI as `aneforge`.
- **Why it matters:** ANEForge exposes the user-space ANE dispatch route; monitoring dispatch events or timing dispatch calls is a credible ANE utilization proxy that does not require private counter register access.

### Finding 3: maderix/ANE — transformer training on ANE via reverse-engineered private APIs
- **Source:** GitHub (maderix/ANE)
- **URL:** https://github.com/maderix/ANE
- **Date:** 2026 (active; follows the earlier maderix Substack M4 ANE series)
- **Summary:** Working implementation training transformers (Stories110M at 91 ms/step, Qwen3-0.6B at 412 ms/step) directly on ANE hardware. Documents 18.6 TFLOPS FP16 / 35.1 TOPS INT8 on M4. Includes dedicated SRAM bandwidth probing benchmarks; actual ANE utilization is ~5–9% of peak under current approach.
- **Why it matters:** The SRAM bandwidth probing methodology is a novel throughput-inference technique worth studying as a counter-free utilization proxy for t3rm1nu55-monitorplus.

### Finding 4: "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" (arXiv 2606.25426)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Microbenchmarking study of M1 AMX: the inner loop is load-issue bound at ~610–680 GFLOPS — under half the load-free rate. A direct-AMX kernel using fine multi-thread panels and weight pre-packing achieves 1.58× geometric mean speedup over Accelerate `BNNSMatMul` across twelve LLM prefill GEMMs.
- **Why it matters:** Confirms no hardware counter path for AMX exists publicly, but documents a timing-based throughput measurement methodology usable as an AMX utilization inference technique.

### Finding 5: jiegec/apple-pmu documents M5 (Hidra) PMU counter events
- **Source:** GitHub (jiegec/apple-pmu — not yet in tracked references; added to references.md this sweep)
- **URL:** https://github.com/jiegec/apple-pmu
- **Date:** 2026
- **Summary:** The jiegec/apple-pmu tool dumps kperf event definitions from `/usr/share/kpep/` plist files. It now includes `as5.md` covering M5 (Hidra). New events in the as5 generation include load data source tracking (`LD_SRC_*`) and PL2 cache events, extending the documented event set beyond M4.
- **Why it matters:** The kperf sidecar in t3rm1nu55-monitorplus needs M5 counter event definitions; jiegec/apple-pmu `as5.md` is the reference to use when adding M5 support.

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
