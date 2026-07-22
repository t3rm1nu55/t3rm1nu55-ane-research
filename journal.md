# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-22 — sweep (8 findings)

### Finding 1: ANE Architecture, Programming, and Performance — first complete public ANE documentation
- **Source:** arXiv / sbryngelson (Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.22283 · Guide: https://ane-guide.readthedocs.io · Repo: https://github.com/sbryngelson/ane-guide
- **Date:** June 21, 2026
- **Summary:** Comprehensive ANE reverse-engineering paper documenting the full stack from silicon up: dispatch below CoreML, E5 compiled program format, weight-compression scheme, kernel driver, firmware, and command protocol. Derived from direct measurement and static decompilation of AppleNeuralEngine.framework, ANECompiler kext, and firmware. A companion web guide and GitHub repo accompany the paper.
- **Why it matters:** Deepest public documentation of ANE internals to date; the kernel driver and firmware command-protocol layers described here are exactly where any ANE hardware counter access would need to hook.

### Finding 2: ANEForge — Python library for direct ANE computation bypassing CoreML
- **Source:** arXiv / sbryngelson (Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.17090 · Repo: https://github.com/sbryngelson/ANEForge
- **Date:** June 12, 2026 (v0.2.0 released June 28, 2026)
- **Summary:** Python package compiling a lazy tensor graph (58 fused + 19 native bridge operators) to a single ANE E5 program, dispatched through the same ANE daemon and kernel-driver stack Apple uses internally, bypassing CoreML entirely. Cross-compiles for M1–M5; runs forward pass, backward pass, and optimizer update on ANE.
- **Why it matters:** First open-source toolchain targeting the ANE's native program format; its dispatch mechanism is the same pathway any future ANE counter instrumentation would need to intercept.

### Finding 3: AMX dual-block structure on M1 documented — "Above the Inner Loop"
- **Source:** arXiv (Georgia Tech)
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Reveals M1's AMX coprocessor has two independent on-chip blocks, exploitable via fine multi-thread panels; a direct-AMX GEMM kernel beats the fastest Accelerate path (BNNS Graph) by 1.17× across 12 LLM prefill shapes. First published account of the M1 second AMX block.
- **Why it matters:** Advances public understanding of AMX microarchitecture; the two-block topology is undocumented by Apple and is directly relevant to any future AMX throughput counter discovery.

### Finding 4: proc_pid_rusage(RUSAGE_INFO_V6).ri_neural_footprint — public per-process ANE memory signal
- **Source:** kennss/SiliconScope v3.0.0 (GitHub)
- **URL:** https://github.com/kennss/SiliconScope (v3.2.1, released July 21, 2026)
- **Date:** June 26, 2026 (v3.0.0)
- **Summary:** SiliconScope v3.0 "Process Inspector" tracks per-process ANE memory via the public call `proc_pid_rusage(pid, RUSAGE_INFO_V6, ...)`, reading field `ri_neural_footprint`. No sudo or private entitlement required for processes owned by the calling user. README explicitly notes this is not compute occupancy — Apple exposes no such metric.
- **Why it matters:** `ri_neural_footprint` is the only documented, public, per-process ANE signal; it reports NE memory held (not compute utilization) but is a viable, zero-privilege proxy for ANE activity; a natural candidate for t3rm1nu55-monitorplus's per-process power section.

### Finding 5: SMEPilot — Arm SME (M4's AMX successor) inference characterization
- **Source:** arXiv
- **URL:** https://arxiv.org/pdf/2606.16332
- **Date:** June 2026
- **Summary:** Characterizes Arm SME (Scalable Matrix Extension — replaces AMX on M4+) for LLM inference. SME is not a universal substitute for vector cores; SMEPilot selects SME/CPU/mixed execution at tile granularity, avoids repeated layout conversion, and gains up to 1.4× throughput improvement over baseline dispatch.
- **Why it matters:** On M4+, the AMX counter discovery problem maps to SME; understanding SME's dispatch and tile model is prerequisite for asking whether M4+ matrix-coprocessor performance events are better-exposed than AMX ones were.

### Finding 6: AsahiLinux/m1n1 — M5 initial support reveals third "M-Core" cluster type
- **Source:** AsahiLinux/m1n1 (commits b72e664, b2d3f5e, d3699d5)
- **URL:** https://github.com/AsahiLinux/m1n1/commit/b2d3f5e
- **Date:** July 10–22, 2026
- **Summary:** m1n1 gained initial support for M5 "Hidra" (T8142, MIDR 0x62/0x63) and M5 Pro/Max "Sotra" (T6050/T6051). M5 Pro/Max introduces a **third core type "M-Core"** (distinct from E-Core and P-Core), used as the boot core. M4 Max "Brava" (T6041) also landed July 22.
- **Why it matters:** Any code iterating IOReport clusters or kperf PMU events with a 2-cluster (E+P) assumption will break on M5 Pro/Max — the M-Core is its own cluster with separate power and performance state.

### Finding 7: AsahiLinux/m1n1 — M3 AMX throttle register offsets in cpufreq driver
- **Source:** AsahiLinux/m1n1 (commit 59dd457)
- **URL:** https://github.com/AsahiLinux/m1n1/commit/59dd457
- **Date:** June 23, 2026
- **Summary:** m1n1 cpufreq driver for M3 (T8122) adds AMX throttle registers at cluster-relative offsets `0x40250` (LLC) and `0x40270` (AMX), adjacent to CPU throttle regs at `0x48400`/`0x48408`.
- **Why it matters:** First published cluster-relative register offsets for M3 AMX throttle control; a hardware anchor for tracing which register-space region AMX performance state is managed from on M3.

### Finding 8: macmon — M4+ IOReport CPU/GPU frequency channels report kHz not Hz
- **Source:** vladkens/macmon (commit 6e90197, fix #57)
- **URL:** https://github.com/vladkens/macmon/commit/6e90197
- **Date:** May 2, 2026
- **Summary:** On M4+ chips, IOReport `CPU Stats` and `GPU Stats` frequency channels report values in **kHz** rather than Hz (the unit M1–M3 use). macmon was applying the wrong ÷1,000,000 divisor, causing 1000× errors; the fix applies ÷1,000 when chip generation is M4+.
- **Why it matters:** Silent compatibility break in IOReport — t3rm1nu55-monitorplus must apply a generation-gated divisor for CPU/GPU frequency sampling or all frequency readings will be off by 1000× on M4+ hosts.

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
