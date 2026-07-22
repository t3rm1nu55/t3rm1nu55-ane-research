# Tracked references

Sources watched by the daily research agent. When new activity appears in any of these, the agent updates `journal.md`.

Organized by category. Each entry has: URL, last-checked date, short description, and (if applicable) "why it matters" for the main `t3rm1nu55-monitorplus` project.

---

## GitHub repositories (reverse engineering & tooling)

| Repo | Description | Last checked | Why it matters |
|---|---|---|---|
| [dougallj/applecpu](https://github.com/dougallj/applecpu) | Firestorm/Icestorm microarchitecture reverse engineering + PMU event documentation | 2026-07-22 | Source of truth for what kperf events exist on M1–M3. Watch for M4/M5 updates. **DORMANT** — last commit July 2023. |
| [hollance/neural-engine](https://github.com/hollance/neural-engine) | ANE reverse engineering, private `_ANEClient` symbol documentation | 2026-07-22 | Any new ANE symbol or counter surface surfaces here first. **DORMANT** — no commits since 2026-04-07. |
| [asahilinux/m1n1](https://github.com/AsahiLinux/m1n1) | Asahi Linux bootloader with extensive M-series hardware register documentation | 2026-07-22 | Asahi folks have the deepest public understanding of M-series hardware registers. **Very active** — M5 bringup, M-Core cluster type, AMX throttle offsets. |
| [vladkens/macmon](https://github.com/vladkens/macmon) | Rust macOS monitor using IOReport | 2026-07-22 | Upstream we vendor the IOReport access pattern from. Watch for channel additions. Note: M4+ IOReport frequency channels report kHz not Hz (commit 6e90197). |
| [dehydratedpotato/socpowerbud](https://github.com/dehydratedpotato/socpowerbud) | Swift IOReport-based SoC power tool | 2026-07-22 | Alternative IOReport reference implementation. **ARCHIVED** — read-only since January 25, 2026. |
| [tlkh/asitop](https://github.com/tlkh/asitop) | Python powermetrics wrapper | 2026-07-22 | Canary for powermetrics sampler changes. **ABANDONED** — last commit Jan 2023. |
| [corellium/linux-m1](https://github.com/corellium/linux-m1) | Corellium Linux-on-M1 port (earlier than Asahi) | 2026-07-22 | Historic reference for some M1 register semantics. Low update rate. |
| [sbryngelson/ANEForge](https://github.com/sbryngelson/ANEForge) | Python library for direct ANE computation bypassing CoreML entirely | 2026-07-22 | First open-source toolchain targeting the ANE's native E5 program format and dispatch stack; companion to the ANE architecture paper. |
| [sbryngelson/ane-guide](https://github.com/sbryngelson/ane-guide) | Open documentation project for ANE architecture, kernel driver, and firmware (readthedocs + arXiv:2606.22283) | 2026-07-22 | Deepest public ANE internals documentation to date — watch for new chip coverage. |
| [kennss/SiliconScope](https://github.com/kennss/SiliconScope) | Native SwiftUI Apple Silicon monitor with sudoless per-process ANE memory tracking | 2026-07-22 | Demonstrates `proc_pid_rusage(RUSAGE_INFO_V6).ri_neural_footprint` as a public per-process ANE signal; actively developed (v3.2.1 as of July 21, 2026). |

## Academic & technical papers

| Source | Description | Last checked | Why it matters |
|---|---|---|---|
| [MIT CSAIL — Jonathan Zhou AMX SB Thesis (2025)](https://commit.csail.mit.edu/papers/2025/Jonathan_Zhou_SB_Thesis.pdf) | Deepest public treatment of AMX performance | 2026-07-22 | Any new MIT CSAIL work on AMX performance characterization. |
| [arXiv — Apple Silicon HPC study (2502.05317)](https://arxiv.org/html/2502.05317v1) | Uses `powermetrics` as ground truth for HPC workload energy measurement | 2026-07-22 | Methodology for paired (workload, energy) measurement. |
| [arXiv — ANE Architecture, Programming, and Performance (2606.22283)](https://arxiv.org/abs/2606.22283) | Full ANE RE paper: datapath, E5 format, kernel driver, firmware, command protocol | 2026-07-22 | New canonical reference for ANE internals. |
| [arXiv — ANEForge: Direct ANE computation (2606.17090)](https://arxiv.org/abs/2606.17090) | Python library compiling tensor graphs to native ANE E5 programs, bypassing CoreML | 2026-07-22 | Watch for new chip support and API surface additions. |
| [arXiv — AMX dual-block GEMM on M1 (2606.25426)](https://arxiv.org/abs/2606.25426) | Documents M1 second AMX block; direct-AMX GEMM kernel beating Accelerate | 2026-07-22 | Best public characterization of M1 AMX microarchitecture. |
| [arXiv — SMEPilot: M4 SME inference (2606.16332)](https://arxiv.org/pdf/2606.16332) | Arm SME (AMX successor in M4+) characterization for LLM inference | 2026-07-22 | Watch for follow-up SME event counter documentation on M4+. |
| [Apple PMU Counter analysis — bugsiki](https://blog.bugsiki.dev/posts/apple-pmu/) | Comprehensive analysis of kperf/kpc counter groups and constraint rules | 2026-07-22 | Authoritative on M1–M4 counter slot allocation and event compatibility. No new posts since January 2026. |
| [maderix — Inside the M4 Apple Neural Engine (Parts 1–3, 2026)](https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-615) | M4 ANE benchmarking, power, and direct access via `_ANEClient`; series complete March 2026 | 2026-07-22 | Series complete; superseded by arXiv:2606.22283 and ANEForge for systematic reference. |
| [ibireme kperf gist](https://gist.github.com/ibireme/173517c208c7dc333ba962c1f0d67d12) | Canonical Objective-C implementation of kperf/kpc counter access | 2026-07-22 | Reference implementation we'd model our own kperf FFI against. |

## Mailing lists & discussion venues

| Source | Description | Last checked | Why it matters |
|---|---|---|---|
| [LKML — Asahi PMU patchset (Marc Zyngier)](https://lkml.kernel.org/lkml/20220208185604.1097957-1-maz@kernel.org/T/) | Upstream Linux PMU driver for Apple M1 | 2026-07-22 | Any follow-up patches for M2/M3/M4/M5 PMU support in Linux. **No new patches found** after April 2026; M3/M4 PMU still unresolved in kernel. |
| [Apple Developer Forums](https://developer.apple.com/forums/) | Occasional hints from DTS about private framework deprecations | 2026-07-22 | Watch for kperf/IOReport/ES deprecation announcements. |

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
