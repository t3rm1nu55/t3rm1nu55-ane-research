# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-08-02 — sweep (6 findings)

### Finding 1: Bryngelson — ANE Architecture, Programming, and Performance (arXiv:2606.22283)
- **Source:** arXiv · sbryngelson/ane-guide
- **URL:** https://arxiv.org/abs/2606.22283 · https://github.com/sbryngelson/ane-guide
- **Date:** June 21, 2026
- **Summary:** Spencer Bryngelson (Georgia Tech) published a comprehensive reverse-engineered reference covering the ANE datapath and roofline, the dispatch route below CoreML, the compiler and on-disk E5 program format, the weight-compression scheme, and the kernel driver/firmware/command protocol — derived from direct measurement and static analysis of private Apple binaries. A web edition is at ane-guide.readthedocs.io.
- **Why it matters:** Documents the full dispatch path from user space to ANE hardware including kernel driver structs and firmware command protocol — the necessary precondition for instrumenting ANE utilization from outside CoreML.

### Finding 2: ANEForge — Python direct ANE dispatch, bypassing CoreML (arXiv:2606.17090)
- **Source:** arXiv · sbryngelson/ANEForge
- **URL:** https://arxiv.org/abs/2606.17090 · https://github.com/sbryngelson/ANEForge
- **Date:** June 12, 2026
- **Summary:** ANEForge is a Python package that compiles a lazy tensor graph into ANE programs and dispatches them via the kernel driver stack, entirely bypassing CoreML. It supports 58 fused operators, cross-compiles to 28 ANE targets (M1–M5), and runs forward + backward + optimizer steps directly on the ANE with a ~90 µs per-program dispatch latency against a 70 µs hardware floor. A companion project, sbryngelson/whisper-aneforge, ports Whisper to the ANE.
- **Why it matters:** The ANEForge dispatch path (kernel driver calls, program structs, command protocol) is now working open code — the primary source to study for injecting utilization monitoring into the ANE call stack.

### Finding 3: AMX inner-loop characterization — M1 is load-issue bound; M4+ switched to ARM SME (arXiv:2606.25426)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 24, 2026
- **Summary:** Deyvik Bhan (Georgia Tech) reverse-engineers the M1 AMX inner loop via microbenchmark: single-thread throughput falls from ~1400 GFLOPS peak to 610–680 GFLOPS when any operand load interleaves with FMA32 (load-issue bound). The paper also notes that M4+ replaced AMX with ARM's Scalable Matrix Extension (SME), which has a documented ARM architecture spec including standard PMU events.
- **Why it matters:** The shift to ARM SME on M4+ is significant — SME-specific PMU events may be accessible via kperf on M4+, unlike proprietary AMX which has no documented event codes.

### Finding 4: kperf/kpc programmable counters confirmed working on M4 Pro (arXiv:2606.27098)
- **Source:** arXiv
- **URL:** https://arxiv.org/abs/2606.27098
- **Date:** June 25, 2026
- **Summary:** Alpay and Başaran use the private kperf/kpc interface on a 14-core M4 Pro to program both fixed counters (cycles, instructions) and configurable counters (L1D load misses, L1D refills, L2-TLB data misses, data table walks). The paper's main subject is GPU cache displacement, but the PMU methodology on M4 Pro is confirmed working.
- **Why it matters:** Confirms the kperf private API works on M4 Pro with configurable counter programming — provides event codes for cross-reference against the kperf sidecar's M4 configuration.

### Finding 5: Eppie/cpu_counter — new single-header Apple Silicon PMU library (GitHub)
- **Source:** GitHub
- **URL:** https://github.com/Eppie/cpu_counter
- **Date:** June 13, 2026
- **Summary:** A new single-header C++ library for PMU counter access on macOS Apple Silicon via the private kperf API, with a bundled demo lab that demonstrates each hardware event with curated workloads. Created June 2026; appears to be a cleaner, more self-contained implementation than the ibireme kperf gist.
- **Why it matters:** New reference implementation for kperf/kpc counter access with validated counter-to-event mappings worth cross-checking against the kperf sidecar.

### Finding 6: m1n1 — AMX system registers gated behind apple_sysregs_unlocked (AsahiLinux/m1n1)
- **Source:** AsahiLinux/m1n1
- **URL:** https://github.com/AsahiLinux/m1n1/commit/001637fab15f712d84dbad87c8e7d32d4d7594fd
- **Date:** June 18, 2026
- **Summary:** Two m1n1 commits gate AMX system-register reads/writes behind an `apple_sysregs_unlocked` flag in both the hypervisor and proxyclient layers, confirming AMX has Apple-proprietary system registers requiring explicit privilege escalation to access.
- **Why it matters:** Any attempt to read AMX activity counters from macOS would face the same access gate — useful context for understanding what kernel-level privilege the main project's kperf sidecar would need for AMX.

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
