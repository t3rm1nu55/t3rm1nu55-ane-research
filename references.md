# Tracked references

Sources watched by the daily research agent. When new activity appears in any of these, the agent updates `journal.md`.

Organized by category. Each entry has: URL, last-checked date, short description, and (if applicable) "why it matters" for the main `t3rm1nu55-monitorplus` project.

---

## GitHub repositories (reverse engineering & tooling)

| Repo | Description | Last checked | Why it matters |
|---|---|---|---|
| [dougallj/applecpu](https://github.com/dougallj/applecpu) | Firestorm/Icestorm microarchitecture reverse engineering + PMU event documentation | 2026-09-06 | Source of truth for what kperf events exist on M1–M3. Watch for M4/M5 updates. |
| [hollance/neural-engine](https://github.com/hollance/neural-engine) | ANE reverse engineering, private `_ANEClient` symbol documentation | 2026-09-06 | Any new ANE symbol or counter surface surfaces here first. |
| [asahilinux/m1n1](https://github.com/AsahiLinux/m1n1) | Asahi Linux bootloader with extensive M-series hardware register documentation | 2026-09-06 | Asahi folks have the deepest public understanding of M-series hardware registers. |
| [vladkens/macmon](https://github.com/vladkens/macmon) | Rust macOS monitor using IOReport | 2026-09-06 | Upstream we vendor the IOReport access pattern from. Watch for channel additions. |
| [dehydratedpotato/socpowerbud](https://github.com/dehydratedpotato/socpowerbud) | Swift IOReport-based SoC power tool | 2026-09-06 | Alternative IOReport reference implementation. Sometimes catches channels macmon misses. |
| [tlkh/asitop](https://github.com/tlkh/asitop) | Python powermetrics wrapper | 2026-09-06 | Canary for powermetrics sampler changes (it broke on macOS 13 when Apple removed `bandwidth`). |
| [corellium/linux-m1](https://github.com/corellium/linux-m1) | Corellium Linux-on-M1 port (earlier than Asahi) | 2026-09-06 | Historic reference for some M1 register semantics. Low update rate. |

## Academic & technical papers

| Source | Description | Last checked | Why it matters |
|---|---|---|---|
| [MIT CSAIL — Jonathan Zhou AMX SB Thesis (2025)](https://commit.csail.mit.edu/papers/2025/Jonathan_Zhou_SB_Thesis.pdf) | Deepest public treatment of AMX performance | 2026-09-06 | Any new MIT CSAIL work on AMX performance characterization. |
| [Shahir M A — What actually runs: ANE measurement study (2608.22110)](https://arxiv.org/abs/2608.22110) | Uses ANE memory-controller byte counters to confirm actual ANE execution vs. CPU fallback during LLM inference | 2026-09-06 | Ground-truth ANE utilization technique using hardware byte counters — the measurement primitive we want. |
| [Bryngelson — ANE: Architecture, Programming, and Performance (2606.22283)](https://arxiv.org/abs/2606.22283) | Complete reverse-engineered ANE reference: datapath, dispatch below CoreML, compiler, kernel driver, firmware, command protocol | 2026-09-06 | Most authoritative public ANE internals reference; kernel driver section may expose counter register locations. |
| [Bryngelson — ANEForge (2606.17090)](https://arxiv.org/abs/2606.17090) / [GitHub](https://github.com/sbryngelson/ANEForge) | Python package for direct ANE compute without CoreML, dispatching through the ANE daemon+driver stack | 2026-09-06 | Accessible direct ANE dispatch; same path where memory-controller byte counters would be readable. |
| [Orion — LLM on ANE (2603.06728)](https://arxiv.org/abs/2603.06728) / [GitHub](https://github.com/mechramc/Orion) | End-to-end LLM training/inference bypassing CoreML, with catalog of 20 ANE MIL program constraints | 2026-09-06 | 20-constraint catalog is the precise map of what the ANE will and won't accept. |
| [Bhan — Above the Inner Loop: AMX GEMM (2606.25426)](https://arxiv.org/abs/2606.25426) | Characterizes M1 AMX: dual-block topology, load-issue bound at 610–680 GFLOPS/thread | 2026-09-06 | Confirms two AMX blocks on M1; load-issue bound is the correct throughput model for AMX bandwidth proxies. |
| [arXiv — Apple Silicon HPC study (2502.05317)](https://arxiv.org/html/2502.05317v1) | Uses `powermetrics` as ground truth for HPC workload energy measurement | 2026-09-06 | Methodology for paired (workload, energy) measurement. |
| [Apple PMU Counter analysis — bugsiki](https://blog.bugsiki.dev/posts/apple-pmu/) | Comprehensive analysis of kperf/kpc counter groups and constraint rules | 2026-09-06 | Authoritative on M1–M4 counter slot allocation and event compatibility. |
| [maderix — Inside the M4 Apple Neural Engine (2024)](https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-615) | M4 ANE benchmarking, power, and direct access via `_ANEClient` | 2026-09-06 | Any follow-up from maderix on ANE or successor chips. |
| [ibireme kperf gist](https://gist.github.com/ibireme/173517c208c7dc333ba962c1f0d67d12) | Canonical Objective-C implementation of kperf/kpc counter access | 2026-09-06 | Reference implementation we'd model our own kperf FFI against. |

## Mailing lists & discussion venues

| Source | Description | Last checked | Why it matters |
|---|---|---|---|
| [LKML — Asahi PMU patchset (Marc Zyngier)](https://lkml.kernel.org/lkml/20220208185604.1097957-1-maz@kernel.org/T/) | Upstream Linux PMU driver for Apple M1 | 2026-09-06 | Any follow-up patches for M2/M3/M4/M5 PMU support in Linux. |
| [Apple Developer Forums](https://developer.apple.com/forums/) | Occasional hints from DTS about private framework deprecations | 2026-09-06 | Watch for kperf/IOReport/ES deprecation announcements. |

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
