# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-08 — sweep (6 findings)

### Finding 1: Comprehensive ANE Architecture and Kernel-Driver Reference
- **Source:** arXiv (Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 21, 2026
- **Summary:** Spencer Bryngelson et al. released a ~46-page reverse-engineered reference for the full ANE stack below CoreML: kernel driver, firmware, command protocol, on-disk program format, weight-compression scheme, and compiler internals. Based on direct measurement on Apple Silicon and static analysis of the private runtime, `_ANECompiler`, and kernel extension. Companion GitHub repo: `sbryngelson/ane-guide`.
- **Why it matters:** This is the deepest public account of the ANE kernel driver and firmware — the most promising place to look for hardware telemetry registers or counter-adjacent interfaces that aren't visible from the CoreML or `_ANEClient` layer.

### Finding 2: ANEForge — Direct ANE Dispatch Library Bypassing CoreML
- **Source:** arXiv / GitHub (Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 12, 2026
- **Summary:** ANEForge is a Python library that compiles a lazy tensor graph (58 fused operators, 19 native bridge operators) to a single ANE program and dispatches it directly to hardware without CoreML, using IOSurface-backed fp16 shared memory for zero-copy I/O. CoreML silently falls back to CPU/GPU; ANEForge guarantees ANE execution.
- **Why it matters:** ANEForge enables reliable, repeatable ANE load generation — a prerequisite for calibrating the IOReport power-indirection approach (correlating IOReport Energy Model deltas against known ANE utilization levels).

### Finding 3: maderix Part 3 — Training on ANE + Public maderix/ANE Repo (missed in initial seed)
- **Source:** maderix Substack / GitHub
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b / https://github.com/maderix/ANE
- **Date:** March 7, 2026
- **Summary:** The third part of maderix's M4 ANE series demonstrates training a 109M-parameter transformer from scratch on the ANE — full forward pass, backward pass, gradient computation, and Adam optimizer — using reverse-engineered private `_ANEClient` APIs. Accompanying code was released in the public `maderix/ANE` GitHub repo.
- **Why it matters:** The `maderix/ANE` repo is now a tracked implementation of `_ANEClient` dispatch; it should be added to references and watched for any telemetry or power-counter observations surfaced during training experiments.

### Finding 4: mperf — Open-Source kperf CLI for Apple Silicon (missed in initial seed)
- **Source:** lambdafoo.com blog
- **URL:** https://lambdafoo.com/posts/2026-03-25-mperf-hardware-counters-macos.html
- **Date:** March 25, 2026
- **Summary:** A new `mperf` tool was published — a `perf stat`-style CLI for Apple Silicon wrapping the private kperf/kperfdata frameworks. Documents 2 fixed counters (cycles, instructions) plus 8 configurable counters (10 simultaneous max, no multiplexing), chip-specific event databases at `/usr/share/kpep/` (e.g. `as4.plist` for M4), and portable aliases that resolve to the correct event ID per chip.
- **Why it matters:** `mperf` is a clean reference implementation of the kperf/kperfdata FFI pattern the main project needs for its privileged sidecar; the 10-counter no-multiplex constraint is an important design constraint for counter selection.

### Finding 5: AMX M1 Inner-Loop Microarchitecture Characterization
- **Source:** arXiv (Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Deyvik Bhan characterizes M1 AMX throughput via microbenchmarks: the inner loop is load-issue bound — any operand load interleaved with the FMA32 stream drops single-thread throughput from ~1.38 TFLOPS to ~610–680 GFLOPS. Beats Accelerate on M1 by restructuring the outer tile loop rather than the inner loop.
- **Why it matters:** Establishes AMX microarchitecture ground truth for M1 without hardware counter discovery; confirms no public AMX-specific kperf events are yet known — AMX throughput must still be inferred via benchmark proxy rather than direct counter readout.

### Finding 6: Orion — ANE Utilization Patterns and Constraint Catalog (missed in initial seed)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 2026
- **Summary:** Orion characterizes the ANE's utilization envelope: deep operation graphs (16–64 ops) achieve 94% ANE utilization; documents 20 ANE constraints (14 newly discovered), including MIL IR, memory layout, and I/O restrictions. Provides a programming model that reliably saturates the engine.
- **Why it matters:** Defines "maximum ANE utilization" in operational terms — a calibration baseline for mapping IOReport Energy Model readings to utilization percentages in the power-indirection approach.

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
