# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-04-14 — sweep (3 findings)

### Finding 1: Orion — first open end-to-end ANE LLM training/inference system

- **Source:** arXiv (found via tracked search: `"Apple Neural Engine" AND ("counter" OR "utilization" OR "benchmark")`)
- **URL:** https://arxiv.org/abs/2603.06728 · https://github.com/mechramc/Orion
- **Date:** March 6, 2026 (missed by seed)
- **Summary:** Orion is the first open system for direct ANE-based LLM training and inference, bypassing CoreML entirely via `_ANEClient`/`_ANECompiler` private APIs. A weight-patching technique sidesteps the documented 119-compile-per-process limit, reducing per-step recompilation from 4,200 ms to 494 ms (8.5×). On M4 Max: 170+ tokens/s for GPT-2 124M inference; 110M-parameter transformer trained from scratch in 22 minutes.
- **Why it matters:** Documents the most complete public `_ANEClient` usage pattern yet. The compile-limit bypass and explicit private API invocation are directly relevant to any future ANE utilization-tracking approach in t3rm1nu55-monitorplus.

### Finding 2: jiegec/apple-pmu — versioned kpep counter archive through macOS Tahoe 26.4

- **Source:** GitHub — jiegec/apple-pmu (untracked repo; found via web search for kperf tooling)
- **URL:** https://github.com/jiegec/apple-pmu
- **Date:** Active; last updated March 25, 2026
- **Summary:** Systematically extracts and commits PMU counter definitions from `/usr/share/kpep` across macOS releases, covering A7 through A19 Pro and all M-series chips. The March 25 commit "Refresh counters for tahoe 26.4" adds data for macOS Tahoe 26.4 betas; coverage also includes iOS device disk images. Added to `references.md` as a new tracked source.
- **Why it matters:** The only public, versioned, per-release archive of kpep counter databases. Directly useful for tracking counter additions/removals across macOS releases and for maintaining the kperf sidecar event list.

### Finding 3: macOS Tahoe 26.5 Beta 2 — M5-era kpep counter definitions pending extraction

- **Source:** MacRumors
- **URL:** https://www.macrumors.com/2026/04/13/apple-releases-macos-tahoe-26-5-beta-2/
- **Date:** April 13, 2026
- **Summary:** Apple released macOS Tahoe 26.5 Beta 2 on April 13. M5 Pro and M5 Max hardware shipped in March 2026; 26.5 is the first beta cycle likely to carry M5-specific kpep counter definitions beyond what 26.4 contained. jiegec/apple-pmu has not yet been updated for 26.5 beta data.
- **Why it matters:** macOS beta cycles are when new PMU counter definitions for new chip variants first surface in `/usr/share/kpep`. A "Refresh counters for tahoe 26.5" commit in jiegec/apple-pmu is the earliest signal for any new M5 PMU events.

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
