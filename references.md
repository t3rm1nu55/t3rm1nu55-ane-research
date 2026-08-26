# Tracked references

Sources watched by the daily research agent. When new activity appears in any of these, the agent updates `journal.md`.

Organized by category. Each entry has: URL, last-checked date, short description, and (if applicable) "why it matters" for the main `t3rm1nu55-monitorplus` project.

---

## GitHub repositories (reverse engineering & tooling)

| Repo | Description | Last checked | Why it matters |
|---|---|---|---|
| [dougallj/applecpu](https://github.com/dougallj/applecpu) | Firestorm/Icestorm microarchitecture reverse engineering + PMU event documentation | 2026-08-26 | Source of truth for what kperf events exist on M1–M3. Watch for M4/M5 updates. |
| [hollance/neural-engine](https://github.com/hollance/neural-engine) | ANE reverse engineering, private `_ANEClient` symbol documentation | 2026-08-26 | Any new ANE symbol or counter surface surfaces here first. |
| [asahilinux/m1n1](https://github.com/AsahiLinux/m1n1) | Asahi Linux bootloader with extensive M-series hardware register documentation | 2026-08-26 | Asahi folks have the deepest public understanding of M-series hardware registers. |
| [vladkens/macmon](https://github.com/vladkens/macmon) | Rust macOS monitor using IOReport | 2026-08-26 | Upstream we vendor the IOReport access pattern from. Watch for channel additions. |
| [dehydratedpotato/socpowerbud](https://github.com/dehydratedpotato/socpowerbud) | Swift IOReport-based SoC power tool | 2026-08-26 | Alternative IOReport reference implementation. Sometimes catches channels macmon misses. |
| [tlkh/asitop](https://github.com/tlkh/asitop) | Python powermetrics wrapper | 2026-08-26 | Canary for powermetrics sampler changes (it broke on macOS 13 when Apple removed `bandwidth`). |
| [corellium/linux-m1](https://github.com/corellium/linux-m1) | Corellium Linux-on-M1 port (earlier than Asahi) | 2026-08-26 | Historic reference for some M1 register semantics. Low update rate. |
| [kennss/SiliconScope](https://github.com/kennss/SiliconScope) | SwiftUI ANE/Media Engine monitor using IOReport + proc_pid_rusage | 2026-08-26 | Exposes `ri_neural_footprint` (per-process ANE memory, public API) — see Finding 5, 2026-08-26 sweep. |
| [sbryngelson/ane-guide](https://github.com/sbryngelson/ane-guide) | Comprehensive ANE architecture reference (datapath, compiler, kernel driver) | 2026-08-26 | Most thorough public documentation of ANE internals; companion to arXiv 2606.22283. |

## Academic & technical papers

| Source | Description | Last checked | Why it matters |
|---|---|---|---|
| [MIT CSAIL — Jonathan Zhou AMX SB Thesis (2025)](https://commit.csail.mit.edu/papers/2025/Jonathan_Zhou_SB_Thesis.pdf) | Deepest public treatment of AMX performance | 2026-08-26 | Any new MIT CSAIL work on AMX performance characterization. |
| [arXiv — Apple Silicon HPC study (2502.05317)](https://arxiv.org/html/2502.05317v1) | Uses `powermetrics` as ground truth for HPC workload energy measurement | 2026-08-26 | Methodology for paired (workload, energy) measurement. |
| [Apple PMU Counter analysis — bugsiki](https://blog.bugsiki.dev/posts/apple-pmu/) | Comprehensive analysis of kperf/kpc counter groups and constraint rules | 2026-08-26 | Authoritative on M1–M4 counter slot allocation and event compatibility. |
| [maderix — Inside the M4 Apple Neural Engine (2024–2026)](https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-615) | M4 ANE benchmarking, power, direct `_ANEClient` access, and full training (Parts 1–3) | 2026-08-26 | Part 3 (training) published 2026; any follow-up on M5 ANE or counter exposure. |
| [ibireme kperf gist](https://gist.github.com/ibireme/173517c208c7dc333ba962c1f0d67d12) | Canonical Objective-C implementation of kperf/kpc counter access | 2026-08-26 | Reference implementation we'd model our own kperf FFI against. |
| [arXiv — AMX GEMM characterization (2606.25426)](https://arxiv.org/abs/2606.25426) | Reveals M1 dual-AMX-block structure and load-issue-bound inner-loop behavior | 2026-08-26 | Essential AMX microarchitecture reference; informs what counters would be meaningful. |
| [arXiv — Orion ANE characterization (2603.06728)](https://arxiv.org/abs/2603.06728) | ANE utilization characterization; 20 dispatch constraints cataloged | 2026-08-26 | Baseline for what 94% ANE utilization looks like in power terms. |
| [arXiv — Bryngelson ANE Architecture (2606.22283)](https://arxiv.org/abs/2606.22283) | Full ANE architecture reference including kernel driver | 2026-08-26 | Best public reference for ANE counter register feasibility investigation. |
| [arXiv — kperf M4 GPU cache study (2606.27098)](https://arxiv.org/abs/2606.27098) | kperf/kpc counter methodology demonstrated on M4 Pro | 2026-08-26 | Reference implementation for M4 counter access in our privileged sidecar. |

## Mailing lists & discussion venues

| Source | Description | Last checked | Why it matters |
|---|---|---|---|
| [LKML — Asahi PMU patchset (Marc Zyngier)](https://lkml.kernel.org/lkml/20220208185604.1097957-1-maz@kernel.org/T/) | Upstream Linux PMU driver for Apple M1 | 2026-08-26 | Any follow-up patches for M2/M3/M4/M5 PMU support in Linux. |
| [Apple Developer Forums](https://developer.apple.com/forums/) | Occasional hints from DTS about private framework deprecations | 2026-08-26 | Watch for kperf/IOReport/ES deprecation announcements. |

## Platforms to monitor for new arXiv preprints

Search strings for the daily agent to run:
- `"Apple Silicon" AND ("PMU" OR "performance counter" OR "kperf")`
- `"Apple Neural Engine" AND ("counter" OR "utilization" OR "benchmark")`
- `"AMX" AND "Apple" AND ("counter" OR "microarchitecture")`
- `"M1" OR "M2" OR "M3" OR "M4" AND ("DVFS" OR "power model" OR "energy")`

---

## How to add a new reference

The daily agent appends new references to the appropriate section. Humans can also add entries directly. When adding:
1. Include the URL
2. Include `Last checked: YYYY-MM-DD`
3. Include a 1-line description
4. Include a "why it matters" note for the main `t3rm1nu55-monitorplus` project
5. Commit with `docs: track <source>`
