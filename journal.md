# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-04-30 — sweep (2 findings)

### Finding 1: Asahi Linux 7.0 Progress Report — PMP driver documents Apple Silicon power-state coprocessor
- **Source:** Asahi Linux blog
- **URL:** https://asahilinux.org/2026/04/progress-report-7-0/
- **Date:** April 26, 2026
- **Summary:** Developer `chaos_princess` contributed a Power Management Processor (PMP) driver to the Asahi Linux kernel tree. PMP is a dedicated Apple Silicon coprocessor that receives power-state reports from all SoC blocks — CPU, GPU, and by extension ANE — via a shared memory region; Linux drivers now coordinate PMGR domain state through it, saving ~0.5 W idle on M1 Pro. The m1n1/Linux-side driver code constitutes public reverse-engineering documentation of the PMP shared-memory protocol.
- **Why it matters:** The PMP shared-memory interface may expose per-block (including ANE) power state at a granularity IOReport's Energy Model channel does not; the Asahi driver is the first public specification of that protocol.

### Finding 2: Apple Silicon `macsmc-power` SMC power driver merges into Linux 7.1 mainline
- **Source:** Phoronix / LWN.net
- **URL:** https://www.phoronix.com/news/Apple-Silicon-Power-Driver-2026 · https://lwn.net/Articles/1059189/
- **Date:** April 2026 (merged in Linux 7.1 merge window)
- **Summary:** Hector Martin's `macsmc-power` driver (~900 LOC), refined by Michael Reeves for upstream, landed in the Linux 7.1 merge window. It exposes AC adapter status, battery capacity, voltage, current, and charging state from the Apple Silicon System Management Controller to userspace — the first time this SMC power-register protocol is canonically documented in mainline Linux.
- **Why it matters:** The SMC power-register protocol is now auditable in mainline kernel source; any SMC registers not surfaced by macOS IOReport are now more discoverable via cross-referencing the Asahi driver against what `powermetrics` reports.

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
