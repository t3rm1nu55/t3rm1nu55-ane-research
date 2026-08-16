# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-16 — sweep (5 findings)

### Finding 1: Comprehensive ANE reverse-engineering reference (A11–M5)
- **Source:** arXiv / GitHub
- **URL:** https://arxiv.org/abs/2606.22283 · https://github.com/sbryngelson/ane-guide · https://ane-guide.readthedocs.io
- **Date:** June 21, 2026 (v2 June 27, 2026)
- **Summary:** "Apple Neural Engine: Architecture, Programming, and Performance" by Spencer H. Bryngelson covers the ANE datapath, dispatch route below CoreML, MIL compiler and on-disk program format, weight-compression scheme, kernel driver, firmware, and command protocol across A11–M5 silicon. It is the most comprehensive public ANE reverse-engineering reference since hollance/neural-engine, now including two generations beyond M4.
- **Why it matters:** The kernel driver and firmware sections are the most likely place to find documented telemetry or performance counter registers within the ANE — exactly the open problem this repo tracks.

### Finding 2: ANEForge — direct Python ANE programming bypassing CoreML
- **Source:** arXiv / PyPI / GitHub
- **URL:** https://arxiv.org/abs/2606.17090 · https://github.com/sbryngelson/ANEForge · https://pypi.org/project/aneforge/
- **Date:** June 12, 2026
- **Summary:** ANEForge compiles a lazy Python tensor graph (58 fused + 19 bridge operators) into a single ANE program without CoreML, achieving ResNet-18 forward pass in 0.33 ms with ~90 µs per-call overhead on Apple Silicon. Published to PyPI.
- **Why it matters:** An independently-maintained CoreML bypass; its implementation of the private ANE dispatch path may expose IOKit properties or timing hooks that could be probed for utilization metrics.

### Finding 3: AMX microarchitecture performance study — M1 AMX inner loop characterized
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" (Deyvik Bhan, Georgia Tech) determines the M1 AMX inner loop is load-issue bound at ~610–680 GFLOPS single-thread and demonstrates a hand-written kernel that beats Accelerate by 1.17× across GPT-2-to-Llama-7B scale prefill GEMMs.
- **Why it matters:** Adds the most precise public characterization to date of AMX throughput saturation and bottleneck structure; useful baseline for any future AMX counter exposure or inference-side utilization proxy work.

### Finding 4: ane-infer — ANE LLM inference via private _ANEClient symbols
- **Source:** GitHub
- **URL:** https://github.com/thebasedcapital/ane-infer
- **Date:** 2026 (exact commit date not pinned in this sweep)
- **Summary:** `ane-infer` implements Apple Neural Engine LLM inference using reverse-engineered private `_ANEClient` symbols, Metal GPU shaders, and hybrid ANE+GPU+CPU scheduling, claiming 32 tok/s and 3.6 TFLOPS fused ANE mega-kernels on Apple Silicon.
- **Why it matters:** Another independent `_ANEClient` user alongside hollance and maderix; dispatch patterns and observed IOKit behavior may surface undocumented properties adjacent to performance telemetry.

### Finding 5: macmon v0.8.2 — per-core IOReport metrics silently broken on M3 Ultra
- **Source:** GitHub (vladkens/macmon)
- **URL:** https://github.com/vladkens/macmon/commit/6919d7781b6c55a6e3bedff83a210435837e1dfe
- **Date:** August 4, 2026
- **Summary:** macmon v0.8.2 restores per-core CPU metrics for M3 Ultra, which had silently regressed due to M3 Ultra's chiplet topology causing IOReport channel name collisions not seen on single-die M3 chips.
- **Why it matters:** M3 Ultra's IOReport layout differs from other M3 variants in ways that silently break per-core metric collection; t3rm1nu55-monitorplus needs verification against M3 Ultra hardware.

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
