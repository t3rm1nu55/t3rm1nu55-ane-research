# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-16 — sweep (5 findings)

### Finding 1: ANEForge — direct Python-to-ANE dispatch without CoreML or entitlements
- **Source:** sbryngelson/ANEForge (GitHub) + arXiv:2606.17090
- **URL:** https://github.com/sbryngelson/ANEForge
- **Date:** June 12, 2026 (paper); June 28, 2026 (v0.2.0 release)
- **Summary:** Spencer Bryngelson (Georgia Tech) published ANEForge, a Python library that compiles lazy tensor graphs into ANE programs and dispatches them through the same ANE daemon and kernel-driver stack as Apple's internal frameworks — no entitlement, no SIP bypass. The library includes 58 fused operators, achieves ~90 µs dispatch latency (near the 70 µs per-program hardware floor), and does training + inference on-ANE. Energy is measured externally via `powermetrics`; no hardware performance counters are exposed.
- **Why it matters:** Proves the ANE daemon path is reachable from an unprivileged user process. ANEForge's Objective-C dispatch layer is a concrete open-source reference for the exact IPC path that monitorplus would need to observe in order to detect ANE dispatch events.

### Finding 2: ane-guide + arXiv:2606.22283 — most comprehensive public ANE architecture documentation
- **Source:** sbryngelson/ane-guide (GitHub) + arXiv:2606.22283
- **URL:** https://github.com/sbryngelson/ane-guide
- **Date:** June 2026
- **Summary:** Companion to ANEForge, the `ane-guide` repo and its paper "Apple Neural Engine: Architecture, Programming, and Performance" (arXiv:2606.22283) document the full ANE hardware architecture, op catalog, memory constraints, and programming model via reverse engineering and decompilation. This is the most thorough public ANE architecture reference published to date, substantially more complete than the hollance/neural-engine notes it supersedes.
- **Why it matters:** Primary architecture reference to consult when scoping any ANE monitoring implementation in monitorplus; confirms the memory hierarchy and dispatch model underlying all current IOReport-based energy monitoring.

### Finding 3: Orion — first open end-to-end ANE training/inference runtime (arXiv:2603.06728)
- **Source:** mechramc/Orion (GitHub) + arXiv:2603.06728
- **URL:** https://github.com/mechramc/Orion
- **Date:** March 2026
- **Summary:** Orion is the first published open system combining direct ANE execution, a compiler pipeline, and stable multi-step training, bypassing CoreML entirely via Apple's private `_ANEClient` and `_ANECompiler` APIs. It extends the public ANE constraint catalog to 20 restrictions (14 newly discovered), and its claimed "94% ANE utilization" figure is derived from throughput benchmarking against peak throughput — not from a hardware counter.
- **Why it matters:** Second independent open implementation of the direct `_ANEClient` dispatch path (alongside hollance's earlier work), confirming the path remains valid on macOS 14/15. Confirms once more that no counter-based utilization metric exists; benchmarking proxies remain the state of the art.

### Finding 4: SiliconScope — `proc_pid_rusage().ri_neural_footprint` for sudoless per-process ANE memory
- **Source:** kennss/SiliconScope (GitHub)
- **URL:** https://github.com/kennss/SiliconScope
- **Date:** Active; v3.2.0 released July 14, 2026
- **Summary:** SiliconScope (native SwiftUI, sudoless) reads the `ri_neural_footprint` field via `proc_pid_rusage()` to expose per-process Neural Engine memory consumption — a different API surface from IOReport Energy Model. The project explicitly acknowledges ANE "utilization" is a power-normalized estimate from IOReport, not true occupancy. Per-process ANE memory via `proc_pid_rusage()` is not currently in the monitorplus feature set.
- **Why it matters:** `ri_neural_footprint` is a new-to-journal, sudoless, per-process API for identifying which processes are consuming ANE resources. This is actionable for monitorplus's process inspector without requiring the privileged sidecar.

### Finding 5: "Above the Inner Loop" AMX paper — two-block structure confirmed via timing probes (arXiv:2606.25426)
- **Source:** arXiv:2606.25426
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** This paper characterizes M1 AMX for LLM prefill GEMM, developing a per-core occupancy probe that reveals Accelerate idles one of the two AMX blocks at certain matrix shapes — confirming the two-block-per-core AMX structure previously only suspected. Occupancy measurement is timing-based (no kperf events); no AMX-specific hardware performance counter events are discovered. The kernel achieves 1.44× throughput over Accelerate at 128-token prefill in llama.cpp.
- **Why it matters:** Confirms that AMX occupancy is measurable via timing probes and establishes the two-block architecture. No new counter API surface, but timing-probe occupancy is the only viable approach absent hardware AMX events — monitorplus AMX tracking would have to follow this pattern.

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
