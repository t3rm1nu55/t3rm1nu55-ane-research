# Tracked references

Sources watched by the daily research agent. When new activity appears in any of these, the agent updates `journal.md`.

Organized by category. Each entry has: URL, last-checked date, short description, and (if applicable) "why it matters" for the main `t3rm1nu55-monitorplus` project.

---

## GitHub repositories (reverse engineering & tooling)

| Repo | Description | Last checked | Why it matters |
|---|---|---|---|
| [dougallj/applecpu](https://github.com/dougallj/applecpu) | Firestorm/Icestorm microarchitecture reverse engineering + PMU event documentation | 2026-07-23 | Source of truth for what kperf events exist on M1–M3. Watch for M4/M5 updates. |
| [hollance/neural-engine](https://github.com/hollance/neural-engine) | ANE reverse engineering, private `_ANEClient` symbol documentation | 2026-07-23 | Any new ANE symbol or counter surface surfaces here first. |
| [asahilinux/m1n1](https://github.com/AsahiLinux/m1n1) | Asahi Linux bootloader with extensive M-series hardware register documentation | 2026-07-23 | Asahi folks have the deepest public understanding of M-series hardware registers. |
| [vladkens/macmon](https://github.com/vladkens/macmon) | Rust macOS monitor using IOReport | 2026-07-23 | Upstream we vendor the IOReport access pattern from. Watch for channel additions. |
| [dehydratedpotato/socpowerbud](https://github.com/dehydratedpotato/socpowerbud) | Swift IOReport-based SoC power tool | 2026-07-23 | Alternative IOReport reference implementation. Sometimes catches channels macmon misses. |
| [tlkh/asitop](https://github.com/tlkh/asitop) | Python powermetrics wrapper | 2026-07-23 | Canary for powermetrics sampler changes (it broke on macOS 13 when Apple removed `bandwidth`). |
| [corellium/linux-m1](https://github.com/corellium/linux-m1) | Corellium Linux-on-M1 port (earlier than Asahi) | 2026-07-23 | Historic reference for some M1 register semantics. Low update rate. |
| [sbryngelson/ANEForge](https://github.com/sbryngelson/ANEForge) | Python package for direct ANE computation without CoreML; companion to arxiv:2606.17090 | 2026-07-23 | Active repo from a group publishing the best current public ANE reverse-engineering work. Watch for new operator support and utilization instrumentation. |
| [sbryngelson/ane-guide](https://github.com/sbryngelson/ane-guide) | Reverse-engineered ANE architecture reference (datapath, compiler, driver, firmware); companion to arxiv:2606.22283 | 2026-07-23 | The canonical public architecture reference for ANE internals; new sections may reveal counter registers. |
| [mechramc/Orion](https://github.com/mechramc/Orion) | First open LLM training/inference runtime on ANE via `_ANEClient`/`_ANECompiler` private APIs | 2026-07-23 | Independent validation of the CoreML-bypass path; may surface new API surface as it matures. |

## Academic & technical papers

| Source | Description | Last checked | Why it matters |
|---|---|---|---|
| [MIT CSAIL — Jonathan Zhou AMX SB Thesis (2025)](https://commit.csail.mit.edu/papers/2025/Jonathan_Zhou_SB_Thesis.pdf) | Deepest public treatment of AMX performance | 2026-07-23 | Any new MIT CSAIL work on AMX performance characterization. |
| [arXiv — Apple Silicon HPC study (2502.05317)](https://arxiv.org/html/2502.05317v1) | Uses `powermetrics` as ground truth for HPC workload energy measurement | 2026-07-23 | Methodology for paired (workload, energy) measurement. |
| [Apple PMU Counter analysis — bugsiki](https://blog.bugsiki.dev/posts/apple-pmu/) | Comprehensive analysis of kperf/kpc counter groups and constraint rules | 2026-07-23 | Authoritative on M1–M4 counter slot allocation and event compatibility. |
| [maderix — Inside the M4 Apple Neural Engine (2024)](https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-615) | M4 ANE benchmarking, power, and direct access via `_ANEClient` | 2026-07-23 | Any follow-up from maderix on ANE or successor chips. |
| [ibireme kperf gist](https://gist.github.com/ibireme/173517c208c7dc333ba962c1f0d67d12) | Canonical Objective-C implementation of kperf/kpc counter access | 2026-07-23 | Reference implementation we'd model our own kperf FFI against. |
| [arXiv — ANEForge (2606.17090)](https://arxiv.org/abs/2606.17090) | Python direct ANE dispatch without CoreML; measures 70 µs dispatch floor | 2026-07-23 | Dispatch-latency-based utilization proxy; companion to ane-guide. |
| [arXiv — ANE Architecture, Programming, and Performance (2606.22283)](https://arxiv.org/abs/2606.22283) | Comprehensive ANE reverse-engineering: datapath, compiler, driver, firmware | 2026-07-23 | Best public ANE architecture reference; any counter register discoveries will appear here first. |
| [arXiv — Above the Inner Loop: AMX GEMM (2606.25426)](https://arxiv.org/abs/2606.25426) | M1 AMX double-block discovery; direct-AMX GEMM outperforms Accelerate | 2026-07-23 | Confirms 2 AMX blocks per P-cluster; relevant to any AMX utilization accounting. |
| [arXiv — Orion: ANE LLM Training (2603.06728)](https://arxiv.org/abs/2603.06728) | First open end-to-end LLM training on ANE via `_ANEClient`/`_ANECompiler` | 2026-07-23 | Independent validation of CoreML-bypass path used by ANEForge; watch for counter instrumentation additions. |
| [ClF3 blog — PMU Event Counters on M3/M4](https://blog.clf3.org/post/pmu-event-counters/) | Documents M3/M4 PMU ESR register width change (16-bit vs 8-bit on M1/M2); M4 kpep file is `as4.plist` | 2026-07-23 | kperf FFI must handle chip-generation-specific ESR field widths to correctly program M3/M4 counters. |

## Mailing lists & discussion venues

| Source | Description | Last checked | Why it matters |
|---|---|---|---|
| [LKML — Asahi PMU patchset (Marc Zyngier)](https://lkml.kernel.org/lkml/20220208185604.1097957-1-maz@kernel.org/T/) | Upstream Linux PMU driver for Apple M1 | 2026-07-23 | Any follow-up patches for M2/M3/M4/M5 PMU support in Linux. |
| [Apple Developer Forums](https://developer.apple.com/forums/) | Occasional hints from DTS about private framework deprecations | 2026-07-23 | Watch for kperf/IOReport/ES deprecation announcements. |

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
