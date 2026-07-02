# Research Journal — ANE/AMX Counter Exposure

Chronological log of findings. Newest entries at the top. Updated daily by an autonomous research agent.

---

## 2026-07-02 — sweep (4 findings)

### Finding 1: "Apple Neural Engine: Architecture, Programming, and Performance" (arXiv 2606.22283)
- **Source:** arXiv preprint (Georgia Tech, Spencer Bryngelson)
- **URL:** https://arxiv.org/abs/2606.22283
- **Date:** June 2026
- **Summary:** Reverse-engineers the ANE from silicon datapath to system interface using hardware measurement on M1 and M5, and static analysis of the private CoreML runtime, compiler, kernel driver, and firmware. Covers the dispatch route below CoreML, the ANE binary format, weight-compression scheme, and the kernel command protocol across A11–A18 and M1–M5 families. Web search summaries reference a `_ANEPerformanceStats` private symbol as a potential hardware counter interface (unverified — arxiv.org blocked by this session's egress policy; treat as a lead, not a confirmed fact).
- **Why it matters:** If `_ANEPerformanceStats` yields readable hardware counters it is the most direct path to real ANE utilization data in t3rm1nu55-monitorplus; warrants symbol inspection on hardware and tracking of any companion GitHub repository.

### Finding 2: "ANEForge: Python for direct computation on the Apple Neural Engine" (arXiv 2606.17090)
- **Source:** arXiv preprint (Georgia Tech, Spencer Bryngelson) — companion to 2606.22283
- **URL:** https://arxiv.org/abs/2606.17090
- **Date:** June 2026
- **Summary:** Python package that compiles a lazy tensor graph (58 fused operators + 19 native bridge operators) directly to a single ANE program, bypassing CoreML entirely. Runs the full forward pass, backward pass, and Adam optimizer updates on the ANE and benchmarks throughput against the CoreML path.
- **Why it matters:** Documents the below-CoreML dispatch protocol and binary format; the dispatch boundary is where timing or power hooks would need to be placed for any future ANE instrumentation path in t3rm1nu55-monitorplus.

### Finding 3: "Above the Inner Loop: Exceeding Accelerate at LLM Prefill GEMM on the M1 AMX" (arXiv 2606.25426)
- **Source:** arXiv preprint (Georgia Tech, Deyvik Bhan)
- **URL:** https://arxiv.org/abs/2606.25426
- **Date:** June 2026
- **Summary:** Empirically characterizes Apple AMX on M1–M3 for LLM prefill GEMM; finds that fine multi-thread panel strategies can saturate M1's second on-chip AMX block, achieving 1.17× speedup over Accelerate's fastest path. Measurement is algorithmic (wall-clock + throughput), not via PMU hardware counters.
- **Why it matters:** Provides updated behavioral constraints on AMX throughput and multi-block parallelism; useful baseline if AMX-specific kperf counter events are ever discovered in a future chip generation.

### Finding 4: "Rigel: Reverse-Engineering the Metal 4.1 Tensor Compute Path on the Apple M4 Max GPU" (arXiv 2606.12765)
- **Source:** arXiv preprint (Ramchand Kumaresan)
- **URL:** https://arxiv.org/abs/2606.12765
- **Date:** June 2026
- **Summary:** Empirically reverse-engineers Apple's Metal 4.1 cooperative_tensor matmul2d path on M4 Max, recovering eleven hardware behaviors not documented in the spec. Key finding: fp8 (E4M3) matmul2d is emulated in software on M4 — throughput is only 0.94× fp16, meaning fp8 is a memory-footprint feature, not a compute-throughput one, on this chip generation.
- **Why it matters:** RE methodology and tooling are directly applicable to ANE characterization on the same chip family; the emulation finding is a useful sanity-check for any fp8 ANE throughput figures.

**Note on GitHub repo coverage:** This session's GitHub MCP access is scoped to this research repo only, and api.github.com is blocked by the session egress policy. Activity on the tracked GitHub repos (dougallj/applecpu, AsahiLinux/m1n1, vladkens/macmon, etc.) was checked via web search only; no new ANE/AMX-relevant commits were surfaced. Direct API checks are deferred to a session with broader GitHub egress.

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
