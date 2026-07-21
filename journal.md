# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-21 — sweep (5 findings)

### Finding 1: "Apple Neural Engine: Architecture, Programming, and Performance" (arxiv 2606.22283)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 2026
- **Summary:** Comprehensive reverse-engineered documentation of the ANE datapath, kernel driver, firmware protocol, and command encoding — the deepest public architectural treatment of any Apple Neural Engine to date. Derives roofline throughput and energy bounds from first principles, establishing how many operations fit within the ANE's pipeline stages. Appears to be a synthesis of the Orion (2603.06728) and maderix lines of work, significantly extended.
- **Why it matters:** This paper describes the ANE command protocol at a level that could underpin a future ANE utilization counter — it is the reference to read before attempting any ANE instrumentation work in t3rm1nu55-monitorplus.

### Finding 2: ANEForge — Python library for direct ANE dispatch (arxiv 2606.17090)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 2026
- **Summary:** Introduces ANEForge, a Python package that compiles and dispatches computations directly to the ANE below CoreML, using reverse-engineered `_ANEClient`/`_ANECompiler` private symbols. Enables raw timing and energy measurement by removing CoreML scheduling overhead, and demonstrates a repeatable utilization measurement methodology.
- **Why it matters:** ANEForge is the most complete public toolchain for direct ANE access; its methodology (dispatch → IOReport energy delta → infer utilization) is directly applicable to how monitorplus could expose an `ane_utilization` signal before hardware counters are found.

### Finding 3: M3/M4 PMU ESR registers use 16-bit event fields (blog.clf3.org)
- **Source:** blog.clf3.org
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** Publication date unconfirmed; content references macOS 15 + M2 Pro, consistent with late 2025 / early 2026
- **Summary:** Documents that M3 and M4 chips extended the kperf/kpc ESR (Event Selection Register) event field from 8 bits to 16 bits, compared to M1 and M2. Event IDs that previously fit in one byte can now encode a wider range of events, meaning any kperf client must branch on chip generation when constructing event masks; the M1/M2 8-bit path will mis-program counters on M3+.
- **Why it matters:** The kperf privileged sidecar in t3rm1nu55-monitorplus currently models M1/M2 event encoding; this confirms a compatibility break on M3/M4 that needs to be handled before the sidecar is tested on newer hardware. **Actionable — opening issue on main project.**

### Finding 4: AsahiLinux/m1n1 adds M5 power-manager register support
- **Source:** AsahiLinux/m1n1 GitHub
- **URL:** https://github.com/AsahiLinux/m1n1/commit/fcaf4765c443d4e7432e470e40c25a1801217f40
- **Date:** 2026-05-05
- **Summary:** m1n1 commit `fcaf4765` extends the PMGR (power-manager) subsystem to cover M4 Pro, M4 Max, A18 Pro, and M5 chip variants (new T-numbers). A related commit (`11934771`) expands `trace_pmp` power-state tracing to T6030, T6031, T6034, T8112, T8122 — covering the full M3/M4 Pro/Max matrix. This is the first open-source power-manager register documentation reaching M5.
- **Why it matters:** Asahi's PMGR work is the best public map of power-domain registers on Apple Silicon; M5 coverage means that when M5 IOReport channels diverge from M4, Asahi's register tables will be the reference for understanding what changed.

### Finding 5: macmon exposes active residency ratios via IOReport (vladkens/macmon)
- **Source:** vladkens/macmon GitHub
- **URL:** https://github.com/vladkens/macmon/commit/3010f1fb7e209c334a1f948ea4386d92e1d761d2
- **Date:** 2026-06-09
- **Summary:** macmon v0.8.0 adds `ecpu_active_ratio`, `pcpu_active_ratio`, and `gpu_active_ratio` fields (plus per-core vectors) derived from `calc_freq`'s third return value: the raw fraction of time a core spent in any non-idle DVFS state, separate from the frequency-scaled effective usage that was previously the only metric. No new IOReport keys are queried; this restructures existing `CPU_FREQ_CORE_SUBG` / `GPUPH` data. No ANE residency metric is added.
- **Why it matters:** monitorplus should mirror this semantic split — reporting both `*_active_ratio` (occupancy) and `*_usage` (frequency-scaled throughput) — to avoid the same misleading throttling-under-load numbers that macmon fixed. The `gpu_active_ratio` pattern is also the model to follow if a future `ane_active_ratio` IOReport key is ever discovered.

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
