# Tracked references

Sources watched by the daily research agent. When new activity appears in any of these, the agent updates `journal.md`.

Organized by category. Each entry has: URL, last-checked date, short description, and (if applicable) "why it matters" for the main `t3rm1nu55-monitorplus` project.

---

## GitHub repositories (reverse engineering & tooling)

| Repo | Description | Last checked | Why it matters |
|---|---|---|---|
| [dougallj/applecpu](https://github.com/dougallj/applecpu) | Firestorm/Icestorm microarchitecture reverse engineering + PMU event documentation | 2026-04-07 | Source of truth for what kperf events exist on M1–M3. Watch for M4/M5 updates. |
| [hollance/neural-engine](https://github.com/hollance/neural-engine) | ANE reverse engineering, private `_ANEClient` symbol documentation | 2026-04-07 | Any new ANE symbol or counter surface surfaces here first. |
| [asahilinux/m1n1](https://github.com/AsahiLinux/m1n1) | Asahi Linux bootloader with extensive M-series hardware register documentation | 2026-04-07 | Asahi folks have the deepest public understanding of M-series hardware registers. |
| [vladkens/macmon](https://github.com/vladkens/macmon) | Rust macOS monitor using IOReport | 2026-04-07 | Upstream we vendor the IOReport access pattern from. Watch for channel additions. |
| [dehydratedpotato/socpowerbud](https://github.com/dehydratedpotato/socpowerbud) | Swift IOReport-based SoC power tool | 2026-04-07 | Alternative IOReport reference implementation. Sometimes catches channels macmon misses. |
| [tlkh/asitop](https://github.com/tlkh/asitop) | Python powermetrics wrapper | 2026-04-07 | Canary for powermetrics sampler changes (it broke on macOS 13 when Apple removed `bandwidth`). |
| [corellium/linux-m1](https://github.com/corellium/linux-m1) | Corellium Linux-on-M1 port (earlier than Asahi) | 2026-04-07 | Historic reference for some M1 register semantics. Low update rate. |
| [mechramc/Orion](https://github.com/mechramc/Orion) | First open end-to-end ANE LLM runtime; bypasses CoreML via `_ANEClient`/`_ANECompiler` | 2026-04-07 | Authoritative reference for the private ANE API surface; delta-compile technique is the current state of the art for direct ANE access. |
| [maderix/ANE](https://github.com/maderix/ANE) | ANE training + inference via reverse-engineered private APIs; INT8 W8A8 support | 2026-04-07 | Companion code to the maderix M4 ANE Substack series; tracks optimizations to direct ANE programming. |
| [verte-zerg/lauka](https://github.com/verte-zerg/lauka) | Apple Silicon PMU counter CLI for recording and comparing kperf events | 2026-04-07 | New reference implementation for kperf tooling; `lauka counters --details` enumerates available events per chip. |

## Academic & technical papers

| Source | Description | Last checked | Why it matters |
|---|---|---|---|
| [MIT CSAIL — Jonathan Zhou AMX SB Thesis (2025)](https://commit.csail.mit.edu/papers/2025/Jonathan_Zhou_SB_Thesis.pdf) | Deepest public treatment of AMX performance | 2026-04-07 | Any new MIT CSAIL work on AMX performance characterization. |
| [arXiv — Apple Silicon HPC study (2502.05317)](https://arxiv.org/html/2502.05317v1) | Uses `powermetrics` as ground truth for HPC workload energy measurement | 2026-04-07 | Methodology for paired (workload, energy) measurement. |
| [Apple PMU Counter analysis — bugsiki](https://blog.bugsiki.dev/posts/apple-pmu/) | Comprehensive analysis of kperf/kpc counter groups and constraint rules | 2026-04-07 | Authoritative on M1–M4 counter slot allocation and event compatibility. |
| [maderix — Inside the M4 Apple Neural Engine, Parts 1–3](https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-c8b) | M4 ANE reverse engineering, benchmarking, and training via `_ANEClient`; Part 3 covers full transformer training | 2026-04-07 | Any follow-up from maderix on ANE or successor chips. |
| [ClF3 — Utilizing PMU Event Counters on Apple M3 and M4](https://blog.clf3.org/post/pmu-event-counters/) | Documents M3/M4 PMU register format differences (64-bit ESR, 16 bits/event) vs M1/M2 (8 bits/event); 100 µs PMCR0 constraint | 2026-04-07 | Directly actionable: kperf sidecar needs chip-gen-aware register handling and must account for the ~100 µs PMU enable window on stock macOS. |
| [Orion paper — arxiv 2603.06728](https://arxiv.org/abs/2603.06728) | Academic characterization of ANE for LLM inference and training; delta-compile technique; M4 Max results | 2026-04-07 | Most rigorous public characterization of the ANE private API surface; companion to mechramc/Orion. |
| [ibireme kperf gist](https://gist.github.com/ibireme/173517c208c7dc333ba962c1f0d67d12) | Canonical Objective-C implementation of kperf/kpc counter access | 2026-04-07 | Reference implementation we'd model our own kperf FFI against. |

## Mailing lists & discussion venues

| Source | Description | Last checked | Why it matters |
|---|---|---|---|
| [LKML — Asahi PMU patchset (Marc Zyngier)](https://lkml.kernel.org/lkml/20220208185604.1097957-1-maz@kernel.org/T/) | Upstream Linux PMU driver for Apple M1 | 2026-04-07 | Any follow-up patches for M2/M3/M4/M5 PMU support in Linux. |
| [Apple Developer Forums](https://developer.apple.com/forums/) | Occasional hints from DTS about private framework deprecations | 2026-04-07 | Watch for kperf/IOReport/ES deprecation announcements. |

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
