# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-04-11 — sweep (3 findings)

### Finding 1: Orion — first systematic ANE programming characterization
- **Source:** arXiv (2603.06728)
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** 2026-03-06
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" maps 20 restrictions on MIL IR programs (14 previously undocumented) and shows that deep operation graphs (16–64 ops) achieve 94% ANE utilization on M4 Max. A per-process ANE compiler bug silently drops compilations after ~119 attempts. MIT-licensed open-source code accompanies the paper.
- **Why it matters:** Orion's constraint catalog is the best public characterization of ANE programmability to date; the ~119-compilation limit is an operational hazard for any tool (including monitorplus) that exercises the ANE repeatedly via private APIs.

### Finding 2: maderix Part 3 + maderix/ANE repo — open-source ANE training code
- **Source:** maderix Substack / GitHub maderix/ANE
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b / https://github.com/maderix/ANE
- **Date:** 2026-03-07 (post) / 2026-03-10 (last repo commit)
- **Summary:** Part 3 demonstrates full transformer training (forward + backward pass, gradient, Adam optimizer) directly on M4 ANE via reverse-engineered `_ANEClient`/`_ANECompiler` private APIs, achieving 1.78 TFLOPS sustained (11.2% of peak) at 9.3 ms/step for a 109M-parameter model. The companion GitHub repo contains working Python code under an open license.
- **Why it matters:** Working open-source code using private ANE APIs is now publicly available; maderix/ANE is the most concrete reference for any future ANE instrumentation work in monitorplus and should be added to tracked references.

### Finding 3: XTC — cross-platform harness using Apple's KPerf/KPep on Apple Silicon
- **Source:** arXiv (2512.16512)
- **URL:** https://arxiv.org/abs/2512.16512
- **Date:** 2025-12-18
- **Summary:** XTC is an AI workload benchmarking platform that accesses Apple's undocumented KPerf system interface and the KPep event-translation database at `/usr/share/kpep/` to read hardware performance counters on Apple Silicon — described as the first cross-platform harness to do so alongside x86 and NVIDIA GPU support.
- **Why it matters:** XTC's KPerf/KPep access pattern is a live reference implementation the monitorplus kperf privileged sidecar can study; the `/usr/share/kpep/` database path is a confirmed stable anchor for event name translation across chip generations.

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
