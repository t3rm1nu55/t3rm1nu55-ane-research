# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-09-10 — sweep (5 findings)

> **Note on GitHub repo checks:** The `gh` CLI is not available in this remote execution environment and the GitHub MCP server is scoped to this repo only. Commits to dougallj/applecpu, hollance/neural-engine, AsahiLinux/m1n1, vladkens/macmon, dehydratedpotato/socpowerbud, tlkh/asitop, and corellium/linux-m1 could not be checked this sweep. Web search did not surface new commits to those repos that indicate ANE/AMX counter work. GitHub repo checks should be re-enabled when the environment has broader API access.

---

### Finding 1: ANE architecture fully reverse-engineered and published (Bryngelson 2606.22283)
- **Source:** arXiv + companion guide at ane-guide.readthedocs.io
- **URL:** https://arxiv.org/abs/2606.22283 / https://github.com/sbryngelson/ane-guide
- **Date:** 2026-06-21
- **Summary:** Spencer Bryngelson (Georgia Tech) published the most comprehensive public reverse-engineering of the Apple Neural Engine to date, covering the full dispatch route below CoreML, the kernel driver and firmware command protocol, the weight-compression scheme, the on-disk program format, and the ANE datapath roofline. The work is based on direct measurement on Apple Silicon and static analysis of the private runtime and compiler.
- **Why it matters:** Documents the exact call path below CoreML that any utilization hook must intercept; the kernel driver protocol section is the closest thing to an ANE hardware register map that has been publicly published.

### Finding 2: ANE memory-controller byte counters observed during inference (arXiv 2608.22110)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2608.22110
- **Date:** 2026-08-22
- **Summary:** "What actually runs" (anonymous, 23pp) measures real ANE placement and decode speed by reading the ANE's memory-controller byte counters during inference, distinguishing what the compiler intended from what actually executed on the ANE. Key finding: placement is a property of how a computation is *expressed*, not what it computes (a fused RMSNorm is fully ANE-eligible; its arithmetically identical unfused decomposition is CPU-only).
- **Why it matters:** **This is the first published use of ANE memory-controller byte counters as a utilization signal.** If these counters are readable via IOReport or a sysctl path, they are the ANE utilization metric t3rm1nu55-monitorplus has been waiting for. The paper's measurement methodology needs to be reverse-engineered — this is now the top priority for the research repo.

### Finding 3: Orion — first open system to bypass CoreML via _ANEClient/_ANECompiler (arXiv 2603.06728)
- **Source:** arXiv + GitHub
- **URL:** https://arxiv.org/abs/2603.06728 / https://github.com/mechramc/Orion
- **Date:** 2026-03-06
- **Summary:** Ramchand Kumaresan's Orion paper is the first open end-to-end system combining direct ANE execution, a compiler pipeline, and LLM training/inference in a single native runtime that bypasses CoreML entirely via Apple's private `_ANEClient` and `_ANECompiler` APIs. Achieves 170+ tokens/s for GPT-2 124M on M4 Max and stable transformer training with an 8.5× weight-patching speedup over naive recompilation.
- **Why it matters:** The `_ANEClient` dispatch path Orion documents is the same hook point where a utilization counter could be inserted; Orion's open codebase is a working reference for the private API surface.

### Finding 4: ANEForge — open Python library for direct ANE computation (arXiv 2606.17090)
- **Source:** arXiv + PyPI + GitHub
- **URL:** https://arxiv.org/abs/2606.17090 / https://github.com/sbryngelson/ANEForge
- **Date:** 2026-06-12
- **Summary:** ANEForge (Bryngelson, same author as Finding 1) is a Python package that compiles a lazy tensor graph of 58 fused operators into ANE programs without CoreML. It supports training, LLM decode/prefill, ONNX import, and scientific computing. The open-source implementation exposes the same private dispatch route documented in the companion architecture paper.
- **Why it matters:** ANEForge's open source compiler is a live, auditable implementation of direct ANE dispatch; inspecting how it emits commands to the ANE kernel driver is now the most tractable path to understanding what a utilization hook would need to observe.

### Finding 5: darwin-kperf — safe Rust bindings for Apple PMU kperf/kpc counters
- **Source:** crates.io
- **URL:** https://crates.io/crates/darwin-kperf / https://docs.rs/darwin-kperf
- **Date:** New since last sweep (exact version date TBD)
- **Summary:** A Rust crate ecosystem (darwin-kperf, darwin-kperf-sys, darwin-kperf-events, darwin-kperf-criterion) providing safe bindings for Apple's private kperf.framework and kperfdata.framework, covering M1–M5. The `darwin-kperf-events` sub-crate packages Apple Silicon PMU event definitions. Requires root or `com.apple.private.kernel.kpc` entitlement; no ABI stability guarantee from Apple.
- **Why it matters:** **Directly actionable for t3rm1nu55-monitorplus:** if the darwin-kperf crate's event definitions cover more events than what our kperf sidecar currently programs, we should evaluate adopting it or diffing its event tables against ours. The criterion integration is also useful for benchmarking counter overhead.

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
