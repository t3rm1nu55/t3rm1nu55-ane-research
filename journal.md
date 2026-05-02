# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-02 — sweep (4 findings)

### Finding 1: M3/M4 PMU ESR uses 16-bit event encoding (breaking change vs M1/M2)
- **Source:** ClF3's blog — "Utilizing PMU Event Counters on Apple M3 and M4"
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** ca. early 2026 (no publication timestamp found)
- **Summary:** Documents that M3/M4 PMU ESR registers are 64-bit with 16-bit per-event encoding, vs. 32-bit ESR with 8-bit per-event encoding on M1/M2. Counter width also differs: 48-bit on M1 (bit 47 triggers PMI) vs. 64-bit on M2/M3/M4 (bit 63 triggers PMI). The post goes chip-by-chip and validates with working sample code.
- **Why it matters:** The kperf sidecar in t3rm1nu55-monitorplus must branch on chip generation when encoding event IDs; any M1-derived code assuming 8-bit event slots will silently program wrong counters on M3/M4 hardware.

### Finding 2: verte-zerg/lauka — new minimal PMU counter CLI
- **Source:** GitHub — verte-zerg/lauka
- **URL:** https://github.com/verte-zerg/lauka
- **Date:** January 2026
- **Summary:** Lauka is a minimal Rust/Swift CLI to record Apple Silicon PMU counters and compare commands, built on top of the ibireme kperf API with ergonomic diff output. It merges the `poop` benchmarking UX with the `scoop` kperf library and extends both. Only works on Apple Silicon.
- **Why it matters:** A clean reference implementation of kperf counter recording that postdates ibireme's gist; worth auditing for any counter encoding or constraint-handling improvements we should adopt in our own sidecar.

### Finding 3: Orion paper — most systematic open ANE characterization to date
- **Source:** arXiv 2603.06728 + github.com/mechramc/Orion
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** March 6, 2026 (arXiv submission)
- **Summary:** Kumaresan's Orion bypasses CoreML entirely via `_ANEClient`/`_ANECompiler` private APIs, catalogs 20 ANE compilation constraints, and achieves 170+ tokens/s GPT-2 inference plus stable 1000-step training on M4 Max. Key technique: "delta compilation" patches baked weights on disk between steps to circumvent compile-time weight baking. Companion code is open MIT at github.com/mechramc/Orion.
- **Why it matters:** The 20-constraint catalog is the deepest public documentation of ANE behavioral limits yet; still no hardware counter API exposed — all measurement is wall-clock throughput — but the compiler internals (MIL graph, IOSurface I/O paths) are now publicly documented and could be the surface where telemetry hooks eventually appear.

### Finding 4: maderix Part 3 + maderix/ANE GitHub repo released
- **Source:** maderix Substack + github.com/maderix/ANE
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b
- **Date:** March 7, 2026
- **Summary:** Part 3 completes the M4 ANE series by demonstrating full transformer training (forward + backward pass, Adam optimizer, 109M params) on hardware Apple built exclusively for inference. The companion GitHub repo at github.com/maderix/ANE includes `api_exploration.m`, a direct probe of `_ANEClient`/`_ANECompiler` internals not covered in the writeups.
- **Why it matters:** `api_exploration.m` is the most direct public enumeration of `_ANEClient` API surface to date; if any counter-like telemetry is accessible through that interface, it would surface in that exploration file first.

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
