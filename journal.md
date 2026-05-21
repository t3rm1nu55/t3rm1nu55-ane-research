# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-05-21 — sweep (3 findings)

### Finding 1: m1n1 adds M5 and A18 Pro power-state register support
- **Source:** AsahiLinux/m1n1
- **URLs:** [pmgr M5 commit](https://github.com/AsahiLinux/m1n1/commit/fcaf4765c443d4e7432e470e40c25a1801217f40) · [T8140 initial commit](https://github.com/AsahiLinux/m1n1/commit/0adfe2ba9b942df57ee9503a01109c6929059d77) · [ps-groups parser](https://github.com/AsahiLinux/m1n1/commit/3cb2ebeea5aaa5174a7272d304ff4626568e064d)
- **Date:** 2026-05-06 (commits merged into main)
- **Summary:** Asahi Linux's m1n1 bootloader received three related commits adding initial hardware support for the M5 and A18 Pro (codename T8140, the chip powering MacBook Neo). The core change restructures the power-manager (`pmgr`) to handle a new `ps-groups` register layout replacing the older `ps-regs` scheme, and adds T8140 as a known SoC with E-core/P-core definitions derived from M4-class features. This is the earliest-stage hardware reverse-engineering of M5 power-state registers visible in any public project.
- **Why it matters:** Asahi's pmgr work is typically the precursor to PMU and perf-counter driver work; once power-domain topology is mapped for M5, kperf event discovery usually follows within months.

### Finding 2: `darwin-kperf` — Rust bindings to Apple kperf/kpc (not previously tracked)
- **Source:** crates.io — `darwin-kperf` (first discovery; pub date predates last sweep)
- **URL:** https://crates.io/crates/darwin-kperf
- **Date:** Last updated 2026-02-23 (discovered this sweep; was not in initial seed)
- **Summary:** A Rust crate providing safe bindings to Apple's private `kperf.framework` and `kperfdata.framework`, exposing hardware PMU counters (cycles, instructions retired, cache misses, branch mispredictions) on Apple Silicon M1–M5. A companion crate `darwin-kperf-criterion` integrates with Criterion.rs as a wall-clock-free measurement backend. Requires root or `com.apple.private.kernel.kpc` entitlement; no ABI stability guarantee from Apple.
- **Why it matters:** Directly substitutes for the custom kperf FFI that t3rm1nu55-monitorplus's privileged sidecar would otherwise need to hand-roll; should be evaluated as a vendored dependency or at minimum as a reference FFI layout.

### Finding 3: Orion paper — first academic ANE utilization characterization (not previously tracked)
- **Source:** arXiv (first discovery; pub date predates last sweep)
- **URL:** https://arxiv.org/abs/2603.06728
- **Date:** 2026-03 (arxiv preprint; not captured in initial seed)
- **Summary:** "Orion: Characterizing and Programming Apple's Neural Engine for LLM Training and Inference" is the most detailed public characterization of ANE execution semantics to date. It measures ANE utilization via throughput-timing benchmarks (not hardware counters) and finds that deep operation graphs (16–64 ops) reach 94% ANE saturation. It catalogs 20 ANE constraints (14 newly discovered), including MIL IR, memory, and I/O restrictions, and identifies a compiler limit of ~119 compilations per process before silent failure.
- **Why it matters:** Confirms that throughput-timing is still the state-of-the-art for ANE utilization estimation — no counter surface has been discovered — and the constraint catalog is the best public guide to what the ANE compiler will and will not accept, directly relevant to any future ANE monitoring probe.

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
