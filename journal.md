# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-06-30 — sweep (4 findings)

### Finding 1: Comprehensive ANE reverse-engineering paper documents kernel driver and command protocol (arXiv 2606.22283)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** 2026-06-21
- **Summary:** Spencer H. Bryngelson published "Apple Neural Engine: Architecture, Programming, and Performance" — a reverse-engineered account of the ANE based on direct measurement on Apple silicon and static analysis of the private runtime, compiler, kernel driver, and firmware. It documents the datapath and roofline, the dispatch route below CoreML, the on-disk program format, the weight-compression scheme, and the full kernel driver / firmware / command protocol. A Hacker News thread (https://news.ycombinator.com/item?id=48702825) confirms high community attention.
- **Why it matters:** First paper to document ANE kernel driver internals and IOKit command protocol from systematic reverse engineering — the most likely place utilization hooks would be found if they exist.

### Finding 2: Orion — first complete LLM training runtime on ANE, catalogs 14 new ANE constraints (arXiv 2603.06728)
- **Source:** arXiv + GitHub (mechramc/Orion)
- **URL:** https://arxiv.org/abs/2603.06728 — https://github.com/mechramc/Orion
- **Date:** 2026-03-16
- **Summary:** Ramchand Kumaresan's "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" is the first end-to-end LLM training system on the ANE that bypasses CoreML entirely via `_ANEClient` and `_ANECompiler`. It extends the public ANE constraint catalog to 20 total (14 newly discovered), documents dispatch overhead (~0.095 ms), a per-process compilation limit (~119 compilations), and a 32 MB SRAM performance cliff. Not captured in the April 7 seed despite predating it.
- **Why it matters:** Extends public hardware characterization of ANE behavior; the constraint catalog and dispatch overhead figure are useful baselines for interpreting IOReport Energy Model sampling intervals.

### Finding 3: maderix "Inside the M4 ANE, Part 3: Training" + maderix/ANE GitHub repo
- **Source:** maderix Substack + GitHub (maderix/ANE)
- **URL:** https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b — https://github.com/maderix/ANE
- **Date:** 2026-03-07
- **Summary:** Part 3 of maderix's M4 ANE series demonstrates full forward + backward pass and Adam optimizer running on the ANE without CoreML, accompanied by the new open-source `maderix/ANE` GitHub repo (Objective-C runtime). The repo is a reference implementation of `_ANEClient`, `_ANECompiler`, and `_ANEInMemoryModelDescriptor` usage. Not captured in the April 7 seed which only tracked Part 2.
- **Why it matters:** `maderix/ANE` is now a trackable code artifact for private ANE API surface; the repo provides the most concise working example of direct ANE dispatch below CoreML.

### Finding 4: ClF3 blog documents M3/M4 PMU ESR register encoding change
- **Source:** ClF3's blog
- **URL:** https://blog.clf3.org/post/pmu-event-counters/
- **Date:** 2026 (exact date TBD — not captured in seed)
- **Summary:** "Utilizing PMU Event Counters on Apple M3 and M4" documents that M3 and M4 changed the kperf ESR register format: events are now 16-bit per slot (vs. 8-bit on M1/M2), and performance counter registers are 64-bit with bit 63 signaling the PMI. This is a structural difference from M1/M2 that affects event configuration code directly.
- **Why it matters:** The kperf sidecar in t3rm1nu55-monitorplus must encode events differently on M3/M4; this post is the primary public documentation of that encoding change and should be referenced in any M3/M4 port.

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
