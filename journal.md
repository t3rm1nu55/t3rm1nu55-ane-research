# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-15 — sweep (6 findings)

### Finding 1: Full ANE architecture, programming, and performance paper (Bryngelson et al., Georgia Tech)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 2026
- **Summary:** The most comprehensive public reverse-engineering of the Apple Neural Engine to date. Bryngelson et al. document the ANE datapath, dispatch route below CoreML, compiler and on-disk program format, weight-compression scheme, kernel driver, firmware, and ANE command protocol — derived from direct measurement on Apple Silicon (A11–M5) and static analysis of private Apple frameworks. A companion web guide is at ane-guide.readthedocs.io and GitHub repo `sbryngelson/ane-guide`.
- **Why it matters:** The kernel driver section documents IOKit ANE-specific power management channels in IOReportLegend (`ANE_ADCLK_TRIG`, `ANE_ADHWTRG`, `ANE_ADSWTRG`, `ANE_DITHR_TRIG`, `ANE_PPT_TRIG`, and others) that could be exploitable as utilization proxies in t3rm1nu55-monitorplus.

### Finding 2: ANEForge — Python framework for direct ANE dispatch below CoreML
- **Source:** arXiv / GitHub
- **URL:** https://arxiv.org/abs/2606.17090
- **GitHub:** https://github.com/sbryngelson/ANEForge
- **Date:** June 12, 2026
- **Summary:** Companion to Finding 1. ANEForge dispatches computation to the ANE via its native `aned` daemon and kernel-driver stack — the same path CoreML, MPSGraph, and Espresso use internally — bypassing CoreML's scheduling layer. Exposes 58 fused operators and 19 bridge operators; performance measurements reveal XPC and IOKit dispatch overhead directly. Supports int8/int4/sparse weights and attention.
- **Why it matters:** The dispatch path is now documented at the kernel-driver level; if Apple exposes utilization counters at the XPC/IOKit layer above the kernel, ANEForge's source reveals where they would appear.

### Finding 3: AMX microarchitecture characterization for LLM prefill GEMM on M1
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Deep AMX microarchitecture characterization for LLM prefill GEMM shapes (QKV, FFN up/down, LM head) using hand-coded AMX instructions via `.word` directives with reverse-engineered encodings. Confirms AMX shares L2 cache with the P-cluster; provides the first rigorous public throughput model tied to specific AMX instruction sequences.
- **Why it matters:** Concrete AMX instruction encodings and throughput data; useful context for interpreting what kperf counter events might correspond to AMX activity when probing undocumented event codes.

### Finding 4: maderix — M4 ANE Part 3: Training on ANE (first-ever)
- **Source:** maderix Substack
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** 2026 (post-April; Part 3 was not present in the prior seed)
- **Summary:** Demonstrates training (forward + backward pass) of a 109M-parameter transformer (Stories110M) on M4 ANE — something CoreML does not support. Author cracked the ANE weight blob format and worked around a 119-compile limit, achieving ~96 ms/step at 2.8W with training state kept resident on-device.
- **Why it matters:** Persistent on-device ANE state across training steps implies there may be session-level monitoring hooks at the `aned` dispatch layer; maderix's approach to direct ANE driver access is now 3 parts deep and is the closest public analog to what an ANE utilization probe would need to do.

### Finding 5: macmon exposes IOReport CPU/GPU active residency ratios
- **Source:** vladkens/macmon (GitHub)
- **URL:** https://github.com/vladkens/macmon/commit/3010f1fb7e209c334a1f948ea4386d92e1d761d2
- **Date:** June 9, 2026
- **Summary:** macmon now exposes `ecpu_active_ratio`, `pcpu_active_ratio`, per-core efficiency/performance values, and `gpu_active_ratio` from IOReport residency data. These "active ratios" are raw residency values without frequency scaling, distinct from the pre-existing frequency-scaled effective-usage metrics.
- **Why it matters:** The active residency pattern (IOReport residency channel → per-component active ratio) is directly adoptable in t3rm1nu55-monitorplus for CPU and GPU; it is also the pattern we would follow for ANE if an IOReport ANE active residency channel becomes accessible.

### Finding 6: m1n1 adds M5 "M-Core" support; gates AMX sysregs on apple_sysregs_unlocked
- **Source:** AsahiLinux/m1n1 (GitHub)
- **URL:** https://github.com/AsahiLinux/m1n1/commit/b2d3f5eba7ced3f5f9e00044744d2c4b4f2efeba
- **Date:** June–July 2026
- **Summary:** m1n1 gains initial M5 Pro (T6050/T6051) bring-up, revealing a third core cluster type called "M-Core" (distinct from P-core and E-core) that lacks `SYS_IMP_APL_E_*` registers. Separately, AMX system register writes in the HV are now gated on `apple_sysregs_unlocked`, reflecting that AMX sysregs are not writable on A18 Pro / M4 Pro and newer without an explicit unlock.
- **Why it matters:** M5 M-Core will need new kperf event mappings when kperf M5 support is extended; the AMX sysreg lock-out on M4 Pro/A18 Pro is a hardware access restriction to account for when probing AMX-related counter codes on those chips.

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
