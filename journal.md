# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-24 — sweep (6 findings)

### Finding 1: Full ANE architecture reverse-engineering paper
- **Source:** arXiv [2606.22283]
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 21, 2026
- **Summary:** Spencer Bryngelson published the most comprehensive public ANE architecture document to date, reverse-engineering the datapath, throughput/energy roofline, kernel driver, firmware, command protocol, compiler pipeline, and weight-compression scheme via direct measurement and static analysis of private runtime components. The paper documents the dispatch path below CoreML, the on-disk ANE program format, and the kernel driver/firmware interaction — all components Apple has not publicly specified.
- **Why it matters:** The kernel driver and firmware protocol documentation is the closest the field has come to a path toward real ANE counter or register exposure; reading this paper is the prerequisite for any t3rm1nu55-monitorplus work on ANE telemetry beyond IOReport energy deltas.

### Finding 2: ANEForge — Python direct dispatch to the ANE below CoreML
- **Source:** arXiv [2606.17090]
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 12, 2026
- **Summary:** ANEForge is a Python package compiling lazy tensor graphs into ANE programs dispatched through Apple's private ANE daemon and kernel-driver stack, bypassing CoreML entirely. It reaches native fused operators and supports int8/int4/sparse weights with 58 fused operators and 19 native bridge operators documented.
- **Why it matters:** Provides a working reference implementation for dispatching to the ANE at the daemon/kernel-driver level — the correct layer at which to probe for any exposed telemetry or utilization counters.

### Finding 3: kperf/kpc PMU counter interface confirmed on M4 Pro
- **Source:** arXiv [2606.27098]
- **URL:** https://arxiv.org/abs/2606.27098
- **Date:** June 25, 2026
- **Summary:** A cache-state characterization paper on the M4 Pro uses the kperf/kpc private interface (root required) to program fixed and configurable PMU counters, measuring per-thread L1D cache refill sectors and line sizes; results validated against STREAM 5.10 and BabelStream 5.0. Demonstrates the complete counter setup workflow: framework linkage, event selection, per-thread sampling.
- **Why it matters:** Confirms the kperf/kpc interface is functional on M4 Pro hardware and provides a concrete code-pattern reference for the counter-sampling approach our privileged kperf sidecar would use.

### Finding 4: AMX inner-loop throughput saturation characterized
- **Source:** arXiv [2606.25426]
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** The first public analysis of AMX throughput saturation: microbenchmarks show the M1 AMX inner loop is load-issue bound at ~610–680 GFLOPS (under half the theoretical peak), and gains over Apple's Accelerate/BNNS Graph come from two-block AMX utilization and pre-packed weights rather than a faster inner loop, achieving 1.17× over BNNS on LLM prefill GEMMs.
- **Why it matters:** Documents that AMX power/throughput is nonlinear at saturation — relevant context if we attempt IOReport-power-based AMX utilization inference, since saturation implies the linear power-to-throughput model breaks down.

### Finding 5: m1n1 AMX register gating and PMU counter continuity fix
- **Source:** AsahiLinux/m1n1 commits (July 2026)
- **URL:** https://github.com/AsahiLinux/m1n1/commits/main/
- **Date:** July 9–15, 2026
- **Summary:** Two commits of direct relevance: (1) the hypervisor now gates AMX system register writes behind an `apple_sysregs_unlocked` capability flag (`hv: only write SPRR/GXF/AMX regs if apple_sysregs_unlocked`), actively refining the AMX register access map; (2) a PMU counter continuity bug was fixed — a chicken bit (`CYC_OVRD_DISABLE_WFI_RET`) that suppressed cycle counter behavior on WFI return was cleared, directly affecting PMU counter accuracy across idle transitions.
- **Why it matters:** The AMX register gating work indicates the m1n1 team is actively mapping AMX sysregisters; any new register documented there could include utilization or throughput signals. The PMU counter continuity fix is a correctness baseline any kperf sidecar must account for on M3/M4 hardware.

### Finding 6: macmon v0.8.0 adds fan speed, active residency, and per-core IOReport channels
- **Source:** vladkens/macmon v0.8.0 release (July 24, 2026)
- **URL:** https://github.com/vladkens/macmon/releases/tag/v0.8.0
- **Date:** July 24, 2026
- **Summary:** macmon v0.8.0 adds per-core CPU view, CPU/GPU active residency ratios, fan speed and fan-name metrics (JSON/Prometheus), and a new `stress` command. The fan speed and residency metrics imply new IOReport channel reads not present in earlier versions; v0.8.2 (Aug 4) fixed per-core metrics on M3 Ultra.
- **Why it matters:** New IOReport channels consumed by macmon (residency, fan speed/name) are candidates to track in t3rm1nu55-monitorplus; the M3 Ultra per-core fix also suggests the channel layout differs across Ultra die configurations and our IOReport parser should handle it.

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
