# Tracked references

Sources watched by the daily research agent. When new activity appears in any of these, the agent updates `journal.md`.

Organized by category. Each entry has: URL, last-checked date, short description, and (if applicable) "why it matters" for the main `t3rm1nu55-monitorplus` project.

---

## GitHub repositories (reverse engineering & tooling)

| Repo | Description | Last checked | Why it matters |
|---|---|---|---|
| [dougallj/applecpu](https://github.com/dougallj/applecpu) | Firestorm/Icestorm microarchitecture reverse engineering + PMU event documentation | 2026-07-18 | Source of truth for what kperf events exist on M1–M3. Watch for M4/M5 updates. |
| [hollance/neural-engine](https://github.com/hollance/neural-engine) | ANE reverse engineering, private `_ANEClient` symbol documentation | 2026-07-18 | Any new ANE symbol or counter surface surfaces here first. |
| [asahilinux/m1n1](https://github.com/AsahiLinux/m1n1) | Asahi Linux bootloader with extensive M-series hardware register documentation | 2026-07-18 | Asahi folks have the deepest public understanding of M-series hardware registers. |
| [vladkens/macmon](https://github.com/vladkens/macmon) | Rust macOS monitor using IOReport | 2026-07-18 | Upstream we vendor the IOReport access pattern from. Watch for channel additions. |
| [dehydratedpotato/socpowerbud](https://github.com/dehydratedpotato/socpowerbud) | Swift IOReport-based SoC power tool | 2026-07-18 | Alternative IOReport reference implementation. Sometimes catches channels macmon misses. |
| [tlkh/asitop](https://github.com/tlkh/asitop) | Python powermetrics wrapper | 2026-07-18 | Canary for powermetrics sampler changes (it broke on macOS 13 when Apple removed `bandwidth`). |
| [corellium/linux-m1](https://github.com/corellium/linux-m1) | Corellium Linux-on-M1 port (earlier than Asahi) | 2026-07-18 | Historic reference for some M1 register semantics. Low update rate. |
| [mechramc/Orion](https://github.com/mechramc/Orion) | First open end-to-end ANE training system; bypasses CoreML via `_ANEClient`/`_ANECompiler` | 2026-07-18 | Documents expanded `_ANEClient` API surface; 20-restriction catalog for MIL IR programs. |
| [maderix/ANE](https://github.com/maderix/ANE) | Training neural networks on ANE via reverse-engineered private APIs (companion to Part 3) | 2026-07-18 | First open-source ANE backward-pass + optimizer implementation; surfaces macOS 26 API breakage. |
| [sbryngelson/ane-guide](https://github.com/sbryngelson/ane-guide) | Reverse-engineered reference for ANE architecture, programming, and performance (Georgia Tech) | 2026-07-18 | Most complete public ANE internals documentation; covers dispatch route, compiler, firmware, command protocol. |
| [sbryngelson/ANEForge](https://github.com/sbryngelson/ANEForge) | Python package for direct ANE computation without CoreML | 2026-07-18 | Python-level direct ANE programming; quantifies 70 µs dispatch floor relevant to power-inference sample intervals. |

## Academic & technical papers

| Source | Description | Last checked | Why it matters |
|---|---|---|---|
| [MIT CSAIL — Jonathan Zhou AMX SB Thesis (2025)](https://commit.csail.mit.edu/papers/2025/Jonathan_Zhou_SB_Thesis.pdf) | Deepest public treatment of AMX performance | 2026-07-18 | Any new MIT CSAIL work on AMX performance characterization. |
| [arXiv — Apple Silicon HPC study (2502.05317)](https://arxiv.org/html/2502.05317v1) | Uses `powermetrics` as ground truth for HPC workload energy measurement | 2026-07-18 | Methodology for paired (workload, energy) measurement. |
| [Apple PMU Counter analysis — bugsiki](https://blog.bugsiki.dev/posts/apple-pmu/) | Comprehensive analysis of kperf/kpc counter groups and constraint rules | 2026-07-18 | Authoritative on M1–M4 counter slot allocation and event compatibility. |
| [maderix — Inside the M4 Apple Neural Engine, Parts 1–3 (2024–2026)](https://maderix.substack.com/p/inside-the-m4-apple-neural-engine-615) | M4 ANE benchmarking, power, direct `_ANEClient` access, and training; Part 3 added March 7, 2026 | 2026-07-18 | Part 3 revealed macOS 26 broke the ANE compile API — signals instability risk for monitorplus private API access. |
| [ibireme kperf gist](https://gist.github.com/ibireme/173517c208c7dc333ba962c1f0d67d12) | Canonical Objective-C implementation of kperf/kpc counter access | 2026-07-18 | Reference implementation we'd model our own kperf FFI against. |
| [Orion: Characterizing and Programming Apple's Neural Engine (arXiv:2603.06728)](https://arxiv.org/abs/2603.06728) | First open end-to-end ANE execution and training system, bypassing CoreML | 2026-07-18 | Documents `_ANEClient`/`_ANECompiler` API surface and catalogs 20 MIL IR restrictions. |
| [ANEForge: Python for direct ANE computation (arXiv:2606.17090)](https://arxiv.org/abs/2606.17090) | Python package for direct ANE programming via 58 fused operators, without CoreML | 2026-07-18 | First Python API for direct ANE programming; establishes 70 µs dispatch floor. |
| [Apple Neural Engine: Architecture, Programming, and Performance (arXiv:2606.22283)](https://arxiv.org/abs/2606.22283) | Comprehensive reverse-engineered reference for ANE internals from datapath to firmware | 2026-07-18 | Most complete public ANE documentation; covers dispatch route below CoreML needed for counter access. |
| [Above the Inner Loop: AMX GEMM on M1 (arXiv:2606.25426)](https://arxiv.org/abs/2606.25426) | Confirms dual AMX block on M1; 1.17x over Accelerate fp32 GEMM | 2026-07-18 | Dual AMX block microarchitecture on M1; IOReport may distinguish single- vs dual-block utilization. |
| [clf3 blog: Utilizing PMU Event Counters on Apple M3 and M4](https://blog.clf3.org/post/pmu-event-counters/) | Documents 16-bit event selector width and PMCR0 kernel preemption on M3/M4 | 2026-07-18 | Critical for kperf sidecar on M3/M4: 16-bit event slots + PMCR0 overwritten by kernel every ~100 µs. |

## Mailing lists & discussion venues

| Source | Description | Last checked | Why it matters |
|---|---|---|---|
| [LKML — Apple M-series PMU patchset, now v10 (Nick Chan)](https://lkml.org/lkml/2026/1/1/91) | Upstream Linux PMU driver for Apple M-series, reached v10 in January 2026 | 2026-07-18 | Per-implementation startup code documents M2/M3/M4 PMU register differences in navigable driver code. |
| [Apple Developer Forums](https://developer.apple.com/forums/) | Occasional hints from DTS about private framework deprecations | 2026-07-18 | Watch for kperf/IOReport/ES deprecation announcements. |

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
