# VRAL — Verified Runtime Anomaly Language

VRAL is a domain-specific language for writing **verified decision kernels**: small, auditable, side-effect-free programs that can be compiled to native shared libraries and embedded in any application or firmware.

A VRAL kernel compiles to a `.so` with no heap allocation, no system calls, and constant execution time — guaranteed at compile time, not at runtime.

---

## Why VRAL

Anomaly detection in safety-critical and embedded systems demands properties that general-purpose ML frameworks cannot provide:

| Requirement | General ML | VRAL |
|-------------|-----------|------|
| No dynamic allocation | ✗ | ✓ Enforced by compiler |
| No OS dependencies | ✗ | ✓ Zero syscalls |
| Deterministic output | ✗ | ✓ Verified against reference |
| Constant execution time | ✗ | ✓ Branchless IR |
| Auditable at binary level | ✗ | ✓ LLVM IR is inspectable |
| Embeddable in C/C++/firmware | Painful | ✓ Plain C ABI |

VRAL enforces these properties through four axioms checked at compile time — not runtime assertions, not coding conventions, but axioms that make violations structurally impossible.

---

## The Four Axioms

| Axiom | Guarantee |
|-------|-----------|
| **A1 — flat memory** | No `malloc`, no `free`, no heap. All state is caller-owned and fixed-size. |
| **A_Sys — no syscalls** | The compiled kernel calls only pure LLVM intrinsics (`llvm.sqrt`, `llvm.maxnum`, etc.). No libc, no OS. |
| **A3 — frozen reference** | Detection is always computed against an immutable baseline model `M_F`, never against a self-adapting state. Immune to poisoning. |
| **A4 — branchless** | All conditionals compile to `fcmp + select`. No data-dependent `br` instructions. Execution time is a compile-time constant. |

---

## What's in this repository

| Document | Description |
|----------|-------------|
| [`whitepaper.md`](whitepaper.md) | Full technical whitepaper — language design, guarantees, VIGIL CWRU results |
| [`GRAMMAR.md`](GRAMMAR.md) | Formal EBNF grammar of the VRAL language |
| [`ABI.md`](ABI.md) | Generated `.so` ABI contract |
| [`architecture/no-syscalls.md`](architecture/no-syscalls.md) | Architectural rationale for the no-syscalls constraint |
| [`examples/server_temp.vral`](examples/server_temp.vral) | Minimal example: server thermal anomaly detection |
| [`examples/motor_health.vral`](examples/motor_health.vral) | Industrial example: bearing fault detection (CWRU validated) |

---

## Try it — Compiler as a Service

VRAL is available as a hosted compilation service. Upload a `.vral` spec, receive a compiled `.so` + `.h` ready to embed:

**Demo:** [https://vral-lang.onrender.com](https://vral-lang.onrender.com)

```bash
# Compile a .vral file to a native shared library
curl -X POST https://vral-lang.onrender.com/v1/compile \
  -F "file=@examples/server_temp.vral" \
  -F "target=x86_64-pc-linux-gnu" \
  -o output.zip

# Calibrate a model from CSV data
curl -X POST https://vral-lang.onrender.com/v1/calibrate \
  -F "file=@normal_data.csv" \
  -F "domain=ServerTemp" \
  -o model.json
```

Supported targets: `x86_64-pc-linux-gnu`, `aarch64-linux-gnu`, `thumbv7em-none-eabihf` (Cortex-M4F), `thumbv7em-none-eabi`.

---

## Validated results

**VIGIL CWRU** — bearing fault detection on the Case Western Reserve University Bearing Dataset:

| Domain | Features | FPR (healthy) | IR fault | OR fault | Latency | Footprint |
|--------|----------|--------------|----------|----------|---------|-----------|
| 2D | RMS_DE + RMS_FE | 1.72% | tick 0 | tick 0 | 15.5 ns | 40 B |
| 3D | RMS_DE + RMS_FE + log(κ) | **0.00%** | tick 0 | tick 0 | 19.3 ns | 64 B |

Measured on AMD EPYC 7B13 (Google Cloud). Window size W=1024 samples at 12 kHz (~85 ms/tick).

---

## License

Documentation and examples in this repository are released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
