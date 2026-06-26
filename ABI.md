# VRAL Generated Library ABI

This document specifies the C ABI of shared libraries (`.so`, `.dll`, `.dylib`) produced by the VRAL compiler.

---

## Function signatures

For a domain named `DomainName` with payload type `T[n]`, the compiler generates two exported functions:

```c
/**
 * Execute one monitoring tick.
 *
 * payload   T[n]        Current observation vector
 * mean      T[n]        Model centroid (from calibration)
 * inv_cov   T[n*n]      Inverse covariance matrix, row-major (if model has NxN matrix field)
 * <params>  T           Scalar model parameters, in declaration order
 * state_ptr T[s]        Caller-owned state buffer; read and written each tick.
 *                       Initialize to all zeros before the first tick.
 *                       One buffer per monitored instance (not shared across instances).
 *
 * Returns int8_t        Verdict index: 1 = first verdict, 2 = second verdict, ...
 */
int8_t domainname_transition(T* payload, T* mean, T* inv_cov, T param1, ..., T* state_ptr);

/**
 * Coherence check between two consecutive payloads.
 *
 * prev, next  T[n]      Two consecutive observation vectors
 *
 * Returns int32_t       1 = coherent (ok), 0 = incoherent (payload rejected)
 */
int32_t domainname_coherence(T* prev, T* next);
```

The function name prefix is the domain name lowercased. For `MotorHealthCWRU3D` the prefix is `motorhealthcwru3d`.

---

## Example: VIGIL 3D domain

```c
#include "motor_health_cwru_3d.h"

// Model parameters from calibration (1797 RPM, OAS)
float mean[3]    = { 0.072861f, 0.083779f, 1.014880f };
float inv_cov[9] = {
    30218.8f, -2450.6f,   53.6f,
    -2450.6f, 26754.7f,   81.7f,
       53.6f,    81.7f,  219.1f,
};
float k     = 0.5908f;
float h     = 26.7936f;
float decay = 0.99f;

float state = 0.0f;   // initialize once; reuse between ticks

// Per-tick usage:
float payload[3] = { rms_de, rms_fe, log_kurt_de };
int8_t verdict = motorhealthcwru3d_transition(payload, mean, inv_cov, k, h, decay, &state);
// verdict == 1 → NORMAL  |  verdict == 2 → ALERTA
```

---

## State ownership

State is **caller-owned**, not library-owned. This enables:

- **N concurrent instances** with one library load: allocate N `float state = 0.0f` buffers, one per monitored entity.
- **Persistence across calls**: the state buffer accumulates the CUSUM signal between ticks.
- **Snapshot and replay**: copy the state buffer to checkpoint or replay a detection sequence.
- **Zero initialization**: always initialize state to `0.0f` (all zeros). The library does not zero-initialize on first call.

---

## Verdict encoding

Verdicts are encoded as `int8_t` in declaration order:

```
1 → first verdict in the `verdicts` declaration
2 → second verdict
...
N → N-th verdict
```

For a domain with `verdicts NORMAL ALERTA`, `1` means `NORMAL` and `2` means `ALERTA`.

---

## Axiom guarantees (inspectable in generated LLVM IR)

| Axiom | Verification method |
|-------|---------------------|
| **A1 — no heap** | No `malloc`/`calloc`/`free` calls in IR. Only fixed-size `alloca` for intermediate array staging. |
| **A_Sys — no syscalls** | Only `llvm.sqrt.f32`, `llvm.maxnum.f32`, `llvm.minnum.f32`, `llvm.fabs.f32` in call sites. No libc. |
| **A3 — deterministic** | Given identical inputs (payload, model, state), the function always returns identical outputs. |
| **A4 — branchless** | Zero data-dependent `br` instructions in the IR. All conditionals are `fcmp + select`. Instruction count is constant for a given domain. |

---

## Compiler targets

| Target triple | Architecture | Use case |
|---------------|-------------|---------|
| `x86_64-pc-linux-gnu` | x86-64 Linux | Server, desktop, cloud |
| `aarch64-linux-gnu` | ARM64 Linux | Raspberry Pi 4, Jetson, cloud ARM |
| `thumbv7em-none-eabihf` | Cortex-M4F bare-metal | STM32, nRF52 (hard-float) |
| `thumbv7em-none-eabi` | Cortex-M4 bare-metal | STM32, nRF52 (soft-float) |
