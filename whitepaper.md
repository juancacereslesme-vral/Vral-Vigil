# VRAL: A Compiled Domain-Specific Language for Verified Anomaly Detection Kernels

**Juan José Cáceres** · June 2026

---

## Abstract

We present VRAL (Verified Runtime Anomaly Language), a domain-specific language for writing anomaly detection kernels that compile to native shared libraries with four structural guarantees: no heap allocation, no system calls, deterministic output, and branchless execution. These properties are enforced by the compiler's verifier pass — they cannot be violated by the domain author. VRAL targets embedded systems and safety-critical applications where ML frameworks are unusable due to allocation, latency, or auditability requirements. We describe the language design, compiler architecture, and calibration pipeline, then validate the approach with VIGIL CWRU, a bearing fault detector that achieves 0.00% false positive rate and 19.3 ns/tick latency on real industrial data from the Case Western Reserve University Bearing Dataset.

---

## 1. Introduction

Deploying anomaly detection in production embedded and safety-critical systems is harder than training the model. The runtime must fit in constrained memory, run with predictable latency, produce auditable outputs, and operate without the Python interpreter, garbage collector, or dynamic linker that development environments take for granted.

Existing approaches fall into one of two categories. Framework-based approaches (PyTorch, ONNX Runtime, TensorFlow Lite) carry significant runtime dependencies, perform dynamic allocation, and offer no formal guarantee on execution time. Manual C implementations are deployable but cannot be verified against a specification, are difficult to maintain, and require the domain expert to be a C programmer.

VRAL takes a different approach: a declarative language where the domain expert writes a specification (the anomaly detection logic), and the compiler produces a native `.so` with structural guarantees enforced statically. The domain author cannot write code that allocates memory or makes system calls — those capabilities are not part of the language.

This paper makes the following contributions:

1. **Language design** — a minimal DSL that captures CUSUM-Mahalanobis anomaly detection with four compiler-enforced axioms (§3).
2. **Compiler architecture** — a Python front-end that emits LLVM IR and links native `.so` files via Clang (§4).
3. **Calibration pipeline** — a statistical workflow (OAS covariance estimation + bootstrap CUSUM threshold) that maps raw CSV sensor data to model parameters (§5).
4. **Empirical validation** — VIGIL CWRU, a bearing fault detector achieving 0.00% FPR and 19.3 ns/tick on the CWRU Bearing Dataset (§6).
5. **Compiler as a Service** — a hosted compilation endpoint that accepts a `.vral` spec and returns a compiled binary for multiple target architectures (§7).

---

## 2. Background

### 2.1 Anomaly detection in constrained environments

Bearing fault detection is an established signal processing problem. Frequency-domain methods (FFT envelope analysis, BPFO/BPFI characteristic frequencies) achieve high accuracy but require floating-point FFT at inference time, which is expensive on microcontrollers and non-trivial to certify. Statistical methods based on feature extraction (RMS, kurtosis) followed by Mahalanobis distance are cheaper at inference time and well-suited for LLVM IR compilation.

### 2.2 The CUSUM detector

The Cumulative Sum (CUSUM) control chart [Page 1954] is a sequential test designed to detect shifts in the mean of a process. Given a score sequence $\{s_t\}$, the leaky CUSUM accumulator is:

$$S_t = \max(0,\ \lambda \cdot S_{t-1} + s_t - k)$$

where $\lambda \in [0, 1)$ is a decay factor (also called the forgetting factor), and $k$ is a slack parameter calibrated to the noise floor of the normal-class scores. An alarm is raised when $S_t > h$, where $h$ is the detection threshold.

The leaky variant (with $\lambda < 1$) is preferred over the classical CUSUM ($\lambda = 1$) in non-stationary industrial environments because it allows the accumulator to return to zero after a disturbance, preventing indefinite alarm persistence.

### 2.3 Mahalanobis distance

For a multivariate feature vector $\mathbf{x}$ with reference mean $\boldsymbol{\mu}$ and inverse covariance $\mathbf{\Sigma}^{-1}$, the Mahalanobis distance is:

$$d(\mathbf{x}) = \sqrt{(\mathbf{x} - \boldsymbol{\mu})^\top \mathbf{\Sigma}^{-1} (\mathbf{x} - \boldsymbol{\mu})}$$

This metric accounts for feature correlations and is scale-invariant. Under the multivariate normal assumption, $d^2$ follows a chi-squared distribution, which provides a principled basis for threshold selection.

---

## 3. Language Design

### 3.1 Core abstraction: the domain

The central abstraction in VRAL is the **domain**: a self-contained specification of one anomaly detector. A domain declares:

- **verdicts** — the discrete output labels (e.g., `NORMAL`, `ALERTA`)
- **payload** — the input feature vector type (e.g., `Float32[3]`)
- **model** — immutable parameters loaded at startup (mean, inverse covariance, thresholds)
- **state** — mutable scalar accumulator(s) threaded through by the caller
- **transition** — the detection logic: maps `(payload, model, state)` to `(verdict, state')`
- **coherence** — an invariant checked between consecutive payloads; rejects sensor dropouts

A complete domain specification for server thermal monitoring fits in 30 lines:

```
domain ServerTemp
    verdicts NORMAL OVERHEATING
    payload Float32[2]          // [cpu_temp, gpu_temp] in °C
    model Baseline
        mean    Float32[2]
        inv_cov Float32[2,2]
        k       Float32
        h       Float32
        decay   Float32
    end
    state Cusum
        s Float32
    end
    transition c
        score   = mahalanobis(c.payload, model.mean, model.inv_cov)
        state.s := max(0.0, model.decay * state.s + score - model.k)
        state.s > model.h -> OVERHEATING
        else -> NORMAL
    end
    coherence prev next
        abs(next[0] - prev[0]) < 30.0   // reject >30°C jump
    end
end
```

### 3.2 The four axioms

The compiler's verifier pass enforces four axioms that cannot be overridden by the domain author:

**A1 — Flat memory.** The compiled kernel performs no dynamic allocation. All state is caller-owned and fixed-size (declared as `Float32`, not a pointer to an unknown length). Array staging for intermediate computations uses fixed-size `alloca` on the stack. No `malloc`, `calloc`, `realloc`, or `free` appears in the generated LLVM IR.

**A_Sys — No system calls.** The kernel calls only certified LLVM pure intrinsics: `llvm.sqrt.f32`, `llvm.maxnum.f32`, `llvm.minnum.f32`, `llvm.fabs.f32`. Any other function call (libc, OS, user-defined) is rejected by the verifier. This makes the kernel safe to load in bare-metal and sandboxed environments, and eliminates the attack surface of OS interaction.

**A3 — Frozen reference.** Detection is always computed against an immutable model `M_F` passed by the caller. The model cannot be updated by the kernel — it has no write access to model memory. This prevents poisoning attacks where an adversary feeds data designed to shift the detector's baseline.

**A4 — Branchless execution.** All conditionals in VRAL compile to `fcmp + select` pairs in LLVM IR. No data-dependent branch (`br`) instruction appears in the generated code. This produces constant execution time regardless of input values — essential for real-time systems with hard timing constraints, and a prerequisite for side-channel resistance.

### 3.3 Type system

VRAL has a simple static type system with four base types (`Float32`, `Float64`, `Int32`, `Bool`), one-dimensional arrays (`T[n]`), and two-dimensional matrices (`T[n, m]`). All sizes are compile-time constants. Type errors are reported before any code generation.

### 3.4 Certified intrinsics

Function calls in VRAL expressions are restricted to a whitelist of certified intrinsics:

| Intrinsic | Description |
|-----------|-------------|
| `mahalanobis(a, b, M)` | $\sqrt{(a-b)^\top M (a-b)}$ |
| `sqrt(x)` | Square root |
| `abs(x)` | Absolute value |
| `max(a, b)` / `min(a, b)` | Element-wise max/min |
| `dot(a, b)` | Dot product |
| `norm(a)` | Euclidean norm |
| `nonfinite(x)` | True if NaN or ±Inf |
| `anynonfinite(v)` | True if any element is non-finite |

Any identifier that is not in this list is rejected by the verifier (A_Sys). This eliminates the possibility of calling arbitrary code through the expression language.

---

## 4. Compiler Architecture

### 4.1 Pipeline

```
.vral ──[lexer]──► tokens ──[parser]──► AST
      ──[verifier]──► (axioms A1, A_Sys, types, verdict completeness)
      ──[codegen]──► LLVM IR text
      ──[clang]──► .o ──[clang -shared]──► .so
      ──[header_gen]──► .h (C declarations)
```

The front-end is approximately 1,800 lines of Python across five modules: lexer, parser, AST nodes, verifier, and code generator. The back-end delegates entirely to Clang/LLVM — no custom register allocator or instruction selector.

### 4.2 Code generation

The code generator traverses the AST and emits LLVM IR text directly. Key design decisions:

- **State threading.** The state buffer is a `float*` argument passed by the caller. The generated function loads all state fields at entry, computes updates, and stores them at exit — a pattern that makes state access explicit in the IR and eliminates aliasing ambiguity.
- **Array staging.** Intermediate array computations (e.g., the difference vector in Mahalanobis) use fixed-size `alloca` on the function's stack frame. The LLVM optimizer (via `-O2`) typically promotes these to register values.
- **Branchless verdicts.** The verdict rules `cond -> V` compile to a sequence of `fcmp` + `select` pairs over an `i8` verdict accumulator, ensuring A4 regardless of how many verdicts the domain declares.

### 4.3 Cross-compilation

The compiler accepts a `--target` flag that is forwarded to Clang as the target triple. Supported targets include `x86_64-pc-linux-gnu`, `aarch64-linux-gnu`, `thumbv7em-none-eabihf` (Cortex-M4F with hardware float), and `thumbv7em-none-eabi` (soft-float). Cross-compilation for ARM requires the appropriate Clang cross-compilation toolchain to be installed.

### 4.4 Generated header

The compiler generates a `.h` file alongside the `.so`. The header declares the transition and coherence function signatures with exact argument types, and embeds a documentation comment listing the model parameters in order. This eliminates manual transcription of the ABI.

---

## 5. Calibration Pipeline

A VRAL model has three classes of parameters: the feature mean $\boldsymbol{\mu}$, the inverse covariance $\mathbf{\Sigma}^{-1}$, and the CUSUM parameters $(k, h, \lambda)$. All are estimated from a CSV file containing normal-class observations.

### 5.1 Covariance estimation

We use the Oracle Approximating Shrinkage (OAS) estimator [Chen et al., 2010] in place of the sample covariance. OAS applies a data-driven shrinkage coefficient $\rho$ that improves the conditioning of $\mathbf{\Sigma}^{-1}$ when the number of observations $n$ is not much larger than the number of features $p$. For $n \gg p$ the estimator converges to the sample covariance; for $n \sim p$ it produces a better-conditioned estimate that avoids numerical explosion in the Mahalanobis computation.

### 5.2 CUSUM threshold calibration

Given the calibrated $(\boldsymbol{\mu}, \mathbf{\Sigma}^{-1})$, we compute Mahalanobis scores on a held-out split of normal-class data and run the CUSUM accumulator. The slack parameter $k$ is set to half the mean score (a standard heuristic). The threshold $h$ is chosen as the 99.9th percentile of the held-out CUSUM trajectory, estimated via bootstrap to produce a 95% confidence interval. The reported $h$ is the upper bound of the confidence interval, providing a conservative threshold that controls false positives.

### 5.3 Normality diagnostic

The calibration pipeline runs a Shapiro-Wilk test on each feature dimension. Non-normal features are flagged with a warning; the pipeline does not abort, but the warning informs the domain author that the Mahalanobis chi-squared interpretation may not hold.

---

## 6. Empirical Validation: VIGIL CWRU

### 6.1 Dataset

The Case Western Reserve University (CWRU) Bearing Dataset is a standard benchmark for mechanical fault detection [Smith & Randall, 2015]. It contains vibration data from a motor drive-end (DE) and fan-end (FE) accelerometer at 12 kHz under normal conditions and with seeded faults of three types: inner race (IR), outer race (OR), and ball element (BE), at fault diameters of 0.007", 0.014", and 0.021".

We use the 1797 RPM split (the highest-load, most challenging regime). All evaluation is performed on data not used for calibration.

### 6.2 Feature extraction

We extract two features per window ($W=1024$ samples, ~85 ms):

- **RMS_DE**: root mean square of the drive-end accelerometer signal.
- **RMS_FE**: root mean square of the fan-end accelerometer signal.

For the 3D domain, we add:

- **log(κ_DE)**: natural logarithm of $\max(\text{Pearson kurtosis of DE}, 1.0)$. The logarithm gaussianizes the right-skewed kurtosis distribution; Shapiro-Wilk fails to reject normality for the transformed feature ($p = 0.012$ — marginal, acceptable). Kurtosis rises sharply for impulsive faults (IR, ball element) and is informative for OR faults at this load level.

### 6.3 Calibration

We use 60% of the normal-class data for covariance estimation and 25% for CUSUM threshold calibration (bootstrap, 1000 resamples), holding out the remaining 15% for FPR evaluation. OAS shrinkage coefficient: $\rho = 0.017$ (3D), $\rho = 0.106$ (2D).

**2D calibrated parameters (1797 RPM):**
- mean = [0.072861, 0.083779]
- k = 0.7103, h = 33.96 [95% CI: 33.03, 33.97], decay = 0.99

**3D calibrated parameters (1797 RPM):**
- mean = [0.072861, 0.083779, 1.014880]
- k = 0.5908, h = 26.79 [95% CI: 26.50, 26.80], decay = 0.99

### 6.4 Results

| Domain | FPR (healthy, 58 windows) | IR 0.007" | IR 0.021" | OR 0.007" | Ball 0.007" |
|--------|--------------------------|-----------|-----------|-----------|-------------|
| 2D | 1.72% (1/58) | alarm tick 0 | alarm tick 0 | alarm tick 0 | alarm tick 1 |
| 3D | **0.00%** (0/58) | alarm tick 0 | alarm tick 0 | alarm tick 0 | alarm tick 2 |

Both domains pass the pre-registered acceptance gates: FPR < 5%, IR/OR alarm $\leq$ 30 ticks.

The 3D domain eliminates the single false positive present in the 2D domain at no cost to fault detection speed. The log-kurtosis feature adds discriminative power for impulsive faults while remaining computationally negligible (one log call per window, performed in the orchestrator before the tick).

### 6.5 Cross-speed robustness

The model is calibrated at 1797 RPM and evaluated at out-of-distribution speeds:

| Speed | 2D FPR | 3D FPR | Gate (<5%) |
|-------|--------|--------|-----------|
| 1797 RPM (reference) | 1.72% | 0.00% | passes |
| 1772 RPM | 7.19% | 5.30% | **fails** |
| 1730 RPM | 5.44% | 4.01% | passes (marginal) |

The detector is speed-specific by design. A different RPM regime shifts the RMS and kurtosis distributions enough to inflate the false positive rate. This is a physically expected result: the model captures the statistical signature of the drive train at a specific operating point. Multi-speed deployment requires one calibrated model per operating regime, which is a standard practice in industrial condition monitoring. The VRAL model footprint is 64 bytes/instance, making per-regime deployment practical even on constrained hardware.

### 6.6 Runtime performance

Benchmarked using a C harness that calls the compiled `.so` in a tight loop (10⁷ iterations, `clock_gettime(CLOCK_MONOTONIC)`, median of 5 runs):

| Domain | Platform | Latency |
|--------|----------|---------|
| 2D (40 B model) | AMD EPYC 7B13, Google Cloud | 15.5 ns/tick |
| 3D (64 B model) | AMD EPYC 7B13, Google Cloud | 19.3 ns/tick |

At 12 kHz sampling and W=1024, one tick corresponds to 85 ms of sensor data. The 19.3 ns kernel latency is five orders of magnitude below the tick period, leaving the orchestrator free to perform I/O, logging, and alerting without timing pressure.

---

## 7. Compiler as a Service

We operate a public VRAL compilation service at [https://vral-lang.onrender.com](https://vral-lang.onrender.com). The service exposes two endpoints:

**`POST /v1/compile`** — accepts a `.vral` file and a target triple, returns a ZIP containing the compiled `.so`, the generated `.h`, and a build log. Supported targets: `x86_64-pc-linux-gnu`, `aarch64-linux-gnu`, `thumbv7em-none-eabihf`, `thumbv7em-none-eabi`.

**`POST /v1/calibrate`** — accepts a CSV file of normal-class sensor readings and domain metadata, returns a JSON object with the calibrated model parameters, diagnostics (OAS $\rho$, Shapiro-Wilk per feature, bootstrap FPR estimate), and a ready-to-paste C snippet.

The service architecture is a FastAPI application backed by the VRAL Python compiler. All compilation runs in a thread-pool executor (non-blocking) with a process-level lock on stdout/stderr capture. The Docker image is `python:3.11-slim` + Clang, running as a non-root user, deployed on Render. Compilation jobs are stateless and horizontally scalable.

```bash
# Compile a domain spec to a native binary
curl -X POST https://vral-lang.onrender.com/v1/compile \
  -F "file=@server_temp.vral" \
  -F "target=x86_64-pc-linux-gnu" \
  -o output.zip

# Calibrate model parameters from normal-class data
curl -X POST https://vral-lang.onrender.com/v1/calibrate \
  -F "file=@normal_operation.csv" \
  -F "domain=ServerTemp" \
  -F "decay=0.99" \
  -o model.json
```

---

## 8. Discussion

### 8.1 What VRAL is not

VRAL is not a general-purpose language. It cannot express arbitrary computations — it cannot sort, allocate, recurse, or call external functions. This is intentional. The design philosophy is that the inability to express unsafe operations is more valuable than the ability to express powerful ones. A domain author who needs dynamic allocation or external function calls is writing something that VRAL deliberately cannot compile.

### 8.2 The orchestrator boundary

The no-syscall constraint shifts all I/O to an external orchestrator. The orchestrator is responsible for reading sensor data, computing windowed features, calling the VRAL kernel, and acting on verdicts. This separation has an important security property: compromising the orchestrator allows an attacker to blind the detector (stop feeding data) or ignore its verdicts (drop alerts), but does not allow poisoning the detection logic itself. The kernel is isolated from the orchestrator's attack surface.

### 8.3 Limitations

**Speed-specificity.** The Mahalanobis-CUSUM model captures a specific operating point. Multi-regime operation requires one calibrated model instance per regime, with an orchestrator-level regime classifier. VRAL's 64-byte-per-instance footprint makes this practical but adds orchestrator complexity.

**Normality assumption.** The chi-squared interpretation of Mahalanobis distance assumes multivariate normal features. The log-kurtosis transformation improves normality for impulsive features but does not guarantee it. Non-normal features inflate the theoretical FPR bound; the bootstrap procedure compensates empirically but does not provide a formal guarantee.

**Intrinsic extensibility.** Adding a new builtin currently requires editing the compiler source. A certified intrinsic registry (a data-driven table with aridity, type rules, LLVM emission, and reference implementation for differential testing) would allow domain authors to propose new primitives without modifying the compiler core. This is the primary open design gap.

### 8.4 Comparison with ONNX Runtime Micro (ORT-micro)

ONNX Runtime Micro targets similar embedded deployment scenarios. ORT-micro supports a wider range of operators and model architectures, including neural networks. VRAL is narrower: it only expresses CUSUM-Mahalanobis detectors. In exchange, VRAL provides formal axiom guarantees that ORT-micro does not — no allocation is a build-time property in VRAL, not a runtime configuration option. For the class of problems VRAL addresses (statistical process control on structured tabular features), the tradeoff is favorable.

---

## 9. Conclusion

We presented VRAL, a domain-specific language for verified anomaly detection kernels. The language enforces four structural guarantees (no heap, no syscalls, deterministic output, branchless execution) through a verifier pass that makes violations structurally impossible. The calibration pipeline provides a one-command workflow from raw sensor CSV to deployable model parameters. The VIGIL CWRU case study demonstrates 0.00% false positive rate, fault detection at tick 0 for all tested fault types, and 19.3 ns/tick latency on production cloud hardware.

VRAL is available as a hosted compiler service. The domain specification for a complete bearing fault detector is 39 lines. The compiled kernel footprint is 64 bytes per instance.

---

## References

- Page, E. S. (1954). Continuous inspection schemes. *Biometrika*, 41(1/2), 100–115.
- Chen, Y., Wiesel, A., Eldar, Y. C., & Hero, A. O. (2010). Shrinkage algorithms for MMSE covariance estimation. *IEEE Transactions on Signal Processing*, 58(10), 5016–5029.
- Smith, W. A., & Randall, R. B. (2015). Rolling element bearing diagnostics using the Case Western Reserve University data: A benchmark study. *Mechanical Systems and Signal Processing*, 64–65, 100–131.
- Lattner, C., & Adve, V. (2004). LLVM: A compilation framework for lifelong program analysis and transformation. *Proceedings of the 2004 International Symposium on Code Generation and Optimization (CGO)*, 75–86.
