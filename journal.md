# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-14 — sweep (5 findings)

### Finding 1: Bryngelson — ANE Architecture, Programming, and Performance (Georgia Tech)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** 2026-06-21
- **Summary:** Comprehensive reverse-engineered account of the Apple Neural Engine by Spencer Bryngelson (Georgia Tech), based on direct measurement and static analysis of Apple's private runtime, compiler, kernel driver, and firmware. Documents the ANE datapath and throughput/energy roofline, the complete dispatch path below CoreML, the on-disk program format, weight-compression scheme, and the kernel driver/firmware/command protocol. Companion reference repository at `sbryngelson/ane-guide`.
- **Why it matters:** First public account deep enough to inform whether ANE utilization is observable below the CoreML abstraction layer — the command protocol section may reveal hooks our kext/daemon sidecar could intercept.

### Finding 2: ANEForge — Direct Python Compute on the ANE (Georgia Tech)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** 2026-06-12
- **Summary:** Spencer Bryngelson (Georgia Tech) releases ANEForge, an open Python package that compiles a lazy tensor graph (58 fused operators) into a native ANE program and dispatches it directly through the ANE daemon and kernel driver, bypassing CoreML entirely. Supports inference, training, and native fused attention; keeps decoder/optimizer state resident across steps. Code at `sbryngelson/ANEForge`.
- **Why it matters:** The bypass technique used here (direct daemon/driver dispatch) is the same layer where utilization observability would need to live; the open codebase is a concrete reference for how dispatch works below CoreML.

### Finding 3: Bhan — M1 AMX Has Two Independent Hardware Blocks
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** 2026-06 (exact day not confirmed)
- **Summary:** Deyvik Bhan (Georgia Tech) shows that the M1 has two independent on-chip AMX blocks and that the M1 AMX inner loop is load-issue bound at ~610–680 GFLOPS single-thread (under half the load-free rate). A hand-written kernel that fills both blocks with fine multi-thread panels beats Apple Accelerate by up to 1.17× on LLM prefill GEMM shapes.
- **Why it matters:** Confirms AMX has two hardware blocks; any future AMX utilization counter would need to account for both. Also tightens the characterization of what "100% AMX utilization" means structurally.

### Finding 4: macmon — Exposes Raw IOReport Active Residency Ratios
- **Source:** vladkens/macmon
- **URL:** https://github.com/vladkens/macmon/commit/3010f1fb7e209c334a1f948ea4386d92e1d761d2
- **Date:** 2026-06-09
- **Summary:** macmon now exposes `*_active_ratio` fields (CPU cluster, per-core, GPU) sourced from IOReport's existing "CPU Core Performance States" and "GPU Stats" channels. The key distinction made explicit: existing `*_usage_ratio` fields are frequency-scaled (a core running at half max frequency contributes half), while the new `*_active_ratio` fields are raw residency (fraction of time the core was not idle, regardless of clock frequency). No new IOReport channels are opened.
- **Why it matters:** The raw residency pattern is directly adoptable in t3rm1nu55-monitorplus's IOReport layer and gives a cleaner per-core idle/active signal for power-inference models that also control for DVFS state.

### Finding 5: m1n1 — M5 Pro (T6050/T6051) Initial Support; New "M-Core" Type
- **Source:** AsahiLinux/m1n1
- **URL:** https://github.com/AsahiLinux/m1n1/commit/b2d3f5eba7ced3f5f9e00044744d2c4b4f2efeba
- **Date:** 2026-07-10
- **Summary:** Asahi adds initial bring-up for T6050 (M5 Pro "Sotra") and T6051 (M5 Max "Sotra C"), introducing new MIDR part constants and a third core taxonomy: "M-Core" (alongside the existing P-Core and E-Core). M-cores do not have `SYS_IMP_APL_E_*` registers and are assigned `features_m4` feature flags — no new PMU register flags are defined yet. M5 Pro boots on core 0, which is an M-Core; SMP secondary startup is not yet functional.
- **Why it matters:** M-cores are a new architectural entity that monitoring tools will need to classify correctly; the absence of E-core-style registers on M-cores means per-cluster PMU counter logic used on M1–M4 will need revision for M5 hardware.

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
