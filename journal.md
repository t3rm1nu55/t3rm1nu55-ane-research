# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-06 — sweep (5 findings)

### Finding 1: Orion — first open end-to-end ANE runtime bypassing CoreML
- **Source:** arxiv.org / github.com/mechramc/Orion
- **URL:** https://arxiv.org/abs/2603.06728 · https://github.com/mechramc/Orion
- **Date:** March 6, 2026 (pre-sweep; missed in seed)
- **Summary:** Orion is the first open system for both LLM inference and training on the Apple Neural Engine without CoreML. It talks to the ANE via private `_ANEClient` and `_ANECompiler` APIs, publishes a catalog of 20 ANE MIL IR constraints (14 newly discovered), and demonstrates stable GPT-2 124M inference at 170 tok/s and 110M-parameter transformer training. The companion GitHub repo contains the full Objective-C runtime, weight converter, and benchmark harness.
- **Why it matters:** Provides a working reference implementation for `_ANEClient`/`_ANECompiler` access patterns and a complete constraint catalog — both directly usable when t3rm1nu55-monitorplus eventually adds ANE state detection or utilization inference.

### Finding 2: clf3.org — M3/M4 PMU event IDs are 16-bit; PMCR0_EL1 has ~100µs kernel-overwrite window
- **Source:** ClF3's blog
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** Unknown (discovered this sweep; new untracked source)
- **Summary:** Documents that M3 and M4 PMUs encode each event in 16 bits inside a 64-bit ESR register, whereas M1/M2 use 8 bits per event — meaning event IDs are not portable across chip generations. Also observes that `SYS_APL_PMCR0_EL1` is repeatedly overwritten by a kernel process; user-space writes typically do not persist longer than ~100 µs.
- **Why it matters:** Our kperf FFI must use chip-generation-specific event encoding constants; the PMCR0 overwrite window sets a hard floor on reliable sampling frequency for any sidecar implementation.

### Finding 3: verte-zerg/lauka — minimal Apple Silicon PMU counter benchmark CLI
- **Source:** GitHub
- **URL:** https://github.com/verte-zerg/lauka
- **Date:** Unknown (discovered this sweep; new untracked source)
- **Summary:** A minimal CLI that records Apple Silicon PMU counters and compares multiple commands across repeated runs, merging the `poop` and `scoop` approaches with added event selection, all-events listing, and warmup support. Documents pairwise and quad incompatibility constraints between specific counter slots.
- **Why it matters:** New reference implementation for kperf counter selection and constraint handling; the documented slot incompatibilities complement bugsiki.dev's constraint tables.

### Finding 4: maderix Part 3 + maderix/ANE repo — backward pass and training confirmed on M4 ANE
- **Source:** maderix Substack / GitHub
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b · https://github.com/maderix/ANE
- **Date:** March 7, 2026 (pre-sweep; missed in seed — journal had Part 2 only)
- **Summary:** Part 3 of the maderix series demonstrates a full backward pass, gradient computation, and Adam optimizer updates on the M4 ANE for a 109M-parameter transformer — the first public training implementation on hardware Apple designed exclusively for inference. The companion `maderix/ANE` GitHub repo contains the implementation code.
- **Why it matters:** Confirms ANE execution model supports stateful, multi-step workloads; the `maderix/ANE` repo is an additional reference alongside `mechramc/Orion` for private ANE API access patterns.

### Finding 5: macmon v0.7.1 — CPU usage always 0% on Ultra chips fixed
- **Source:** vladkens/macmon
- **URL:** https://github.com/vladkens/macmon/releases/tag/v0.7.1
- **Date:** April 15, 2026
- **Summary:** macmon v0.7.1 fixed CPU cluster usage always reporting 0% on M1/M2/M3 Ultra chips (issue #55) caused by incorrect IOReport cluster topology handling for the dual-die Ultra configuration.
- **Why it matters:** t3rm1nu55-monitorplus reads the same IOReport CPU cluster channels; if Ultra chip support is in scope, the root cause of this bug (dual-die topology not being accounted for) applies equally to our implementation.

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
