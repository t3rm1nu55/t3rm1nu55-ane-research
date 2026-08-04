# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-04 — sweep (8 findings)

### Finding 1: Bryngelson — "Apple Neural Engine: Architecture, Programming, and Performance" (arXiv 2606.22283)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 21, 2026
- **Summary:** Comprehensive reverse-engineering of the ANE across A11–A18 and M1–M5: datapath architecture, roofline bounds, below-CoreML dispatch via private `_ANEClient`/`_ANECompiler` APIs, on-disk compiler/program format, weight-compression scheme, kernel driver, firmware, and command protocol — all derived from direct measurement and static binary analysis. Most complete public ANE hardware characterization to date, with per-chip performance tables for M1 through M5.
- **Why it matters:** Documents the kernel driver and command protocol in detail — review for any PMU event or IOReport channel disclosures that could feed into t3rm1nu55-monitorplus ANE power telemetry.

### Finding 2: Bryngelson — "ANEForge: Python for Direct Computation on the Apple Neural Engine" (arXiv 2606.17090)
- **Source:** arXiv / GitHub
- **URL:** https://arxiv.org/abs/2606.17090 · https://github.com/sbryngelson/ANEForge
- **Date:** June 12, 2026
- **Summary:** Companion to 2606.22283. Introduces a Python toolkit that compiles a lazy tensor graph of 58 fused + 19 bridge operators directly into ANE programs, bypassing CoreML entirely using the private `_ANEClient` dispatch path.
- **Why it matters:** The direct dispatch path is the prerequisite for any future ANE utilization instrumentation; ANEForge makes that path reproducible and gives a concrete symbol list to work against.

### Finding 3: Alpay & Basaran — "Residual GPU Cache State on Apple M4 Pro" (arXiv 2606.27098)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.27098
- **Date:** June 25, 2026
- **Summary:** Uses `kperf`/`kpc` (2 fixed + 5 configurable per-thread PMU counters) on an M4 Pro to characterize GPU cache state left after completed GPU commands, discovering the actual L1D refill sector is 64 bytes despite macOS reporting 128-byte cache lines — found via counter measurement, not Apple documentation.
- **Why it matters:** Confirms kperf counter access still works on M4 Pro and demonstrates which counter slots are programmable; concrete model for counter-based undocumented-SoC-behavior discovery applicable to our kperf sidecar.

### Finding 4: Bhan — "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" (arXiv 2606.25426)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 24, 2026
- **Summary:** Microbenchmark analysis of the M1 AMX inner loop showing it is load-issue bound, collapsing single-thread throughput to 610–680 GFLOPS when any operand load interleaves with the FMA32 stream (roughly half the load-free theoretical rate); no inner-loop rearrangement tested can escape this bound.
- **Why it matters:** Advances AMX microarchitectural understanding without counter access — the load-issue bound is the ceiling any future AMX utilization metric must account for.

### Finding 5: macmon — Frida-based libIOReport.dylib analysis characterizes powermetrics ANE subscription
- **Source:** vladkens/macmon
- **URL:** https://github.com/vladkens/macmon/commit/40f4e46
- **Date:** July 23, 2026
- **Summary:** macmon added `research/powermetrics/` containing Frida JavaScript hooks for `libIOReport.dylib` that trace Apple's own powermetrics binary. Documents the exact IOReport subscription used: 3 subscriptions, with the Energy Model covering **136 channels** (ANE, GPU, DRAM, display, media, SRAM, SoC). Sampling sequence — subscription init → sample → delta → interval wait — is now precisely characterized.
- **Why it matters:** Removes ambiguity about which IOReport channels Apple itself uses to sample ANE energy; directly applicable to t3rm1nu55-monitorplus IOReport subscription setup.

### Finding 6: jiegec/apple-pmu — M5 kpep confirmed identical to M4, no ANE/AMX events added (Jul 3)
- **Source:** jiegec/apple-pmu
- **URL:** https://github.com/jiegec/apple-pmu
- **Date:** July 3, 2026
- **Summary:** `as5.plist` (M5 kpep database) is identical to `as4.plist` — no new ANE or AMX performance counter events through M5. M4 did add SME engine counters (`INST_SME_ENGINE_*`, `CORE_WAITING_SME_ENGINE_CYCLE`) but these count ARM SME within CPU cores, not operations on the ANE block or Apple's proprietary AMX.
- **Why it matters:** Definitive negative through M5: Apple has not exposed ANE/AMX events in kperf. The SME counters are for ARM SME, not Apple's proprietary AMX — these two must not be conflated.

### Finding 7: kennss/SiliconScope — M5 Max IOReport channel map + M4 Max AMC Stats regression
- **Source:** kennss/SiliconScope (newly tracked)
- **URL:** https://github.com/kennss/SiliconScope · commit 395f592
- **Date:** July 29, 2026
- **Summary:** Documents M5 Max IOReport rail naming changes (no `EACC_CPU`, `MACC*` for CPU bandwidth, `PMP0` not `PMP`); confirms ANE0/ANE1 Energy Model channels work on M5 Max at expected power levels. Separately: on macOS 26.5.2, `AMC Stats` bandwidth channels on M4 Max are "discoverable but not subscribable" — the PMP fallback path is required.
- **Why it matters:** M5 Max rail names differ from M1–M4; IOReport channel-name tables in t3rm1nu55-monitorplus will need per-generation updates for M5 Max. The M4 Max AMC Stats regression is a live macOS 26 constraint.

### Finding 8: AsahiLinux/m1n1 — AMX control registers at EL2 named and documented (Jun 18)
- **Source:** AsahiLinux/m1n1
- **URL:** https://github.com/AsahiLinux/m1n1/commit/53d4271
- **Date:** June 18, 2026
- **Summary:** Two paired commits gate Apple proprietary sysreg writes behind `apple_sysregs_unlocked`, incidentally documenting the AMX EL2 register set: `SYS_IMP_APL_AMX_CTL_EL2` (enables `AMX_CONFIG_EL1.EN_EL1`), `APVMKEYLO_EL2`, `APVMKEYHI_EL2`, `APSTS_EL12`. This is HV AMX state management, not PMU counter infrastructure.
- **Why it matters:** Extends the ground-truth AMX register name map; `AMX_CTL_EL2` is how the hypervisor enables per-core AMX — a prerequisite for understanding AMX isolation boundaries if counter injection is ever attempted from the hypervisor layer.

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
