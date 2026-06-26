# VRAL No-Syscall Architecture

**Status:** Implemented and verifiable property of the current VRAL compiler.

This document explains the architectural decision to prohibit system calls in VRAL kernels, and the rationale behind it.

---

## 1. The constraint

A compiled VRAL kernel cannot execute operating system calls. The kernel has no native capability to:

- Read or write files
- Open, close, or query network sockets
- List processes, ports, or kernel state
- Modify system configuration
- Load dynamic libraries
- Invoke external binaries
- Access hardware directly

All interaction with the external environment is delegated to the **orchestrator**: an application layer that wraps the VRAL kernel, handles I/O, and acts on verdicts. The kernel receives pre-processed data through caller-owned memory buffers, runs its detection logic, and emits a verdict. It never touches the OS directly.

This separation is structural, not a coding convention. The syscall primitives are absent from the language. It is not "we choose not to use them" — it is "they do not exist here."

---

## 2. Rationale

### 2.1 Auditability

A kernel without syscalls can be fully audited by inspecting its LLVM IR. Every operation is visible: arithmetic, comparisons, memory loads and stores, and calls to LLVM intrinsics. There are no hidden I/O paths. This makes formal verification tractable and simplifies security review.

### 2.2 Portability

A kernel that makes no OS calls runs identically on Linux, macOS, Windows, and bare-metal ARM Cortex-M. The same `.so` compiled for `thumbv7em-none-eabihf` runs on an STM32 microcontroller without modification. OS-specific behavior cannot accidentally creep in.

### 2.3 Isolation from the orchestrator's attack surface

The orchestrator (the layer that feeds data to the kernel and acts on its verdicts) has a classical attack surface: it reads from sensors, writes to logs, opens network connections, and calls external services. Compromising the orchestrator is possible through standard software vulnerabilities.

The no-syscall constraint isolates the kernel from this attack surface. Compromising the orchestrator allows an attacker to blind the kernel (stop feeding it data) or ignore its verdicts (suppress alerts). It does not allow poisoning the kernel's detection logic, modifying its model parameters, or causing the kernel to produce incorrect verdicts on data it does see.

---

## 3. Threat model: the dark profiles

The no-syscall constraint was motivated in part by an analysis of how a system with VRAL's architectural properties could be misused if it had OS access. The four threat profiles below document the attack scenarios that the constraint neutralizes. They are included here because the constraint is only fully understood in light of what it prevents.

### Profile A — The Ghost

**Hypothetical offensive capability:** Immunity to process-based detection signatures.

**Mechanism:** Traditional antivirus tools search for persistent processes with a specific signature (a PID, a binary on disk, an identifiable memory section). A VRAL-based attacker would not exist as a static process but as a cascade of ephemeral capsids. When the detection system targets capsid A, that capsid has already transferred its state to capsid B and ceased to exist. The detector attempts to kill a corpse millions of times per second.

**Why the no-syscall constraint neutralizes it:** The Ghost needs to persist in memory in order to execute actions (open connections, read files, escalate privileges). Without syscalls, the capsid cascade cannot act on the OS. It is invisible but also inert — a ghost that cannot do anything.

### Profile B — The Polymorphic

**Hypothetical offensive capability:** Continuous mutation that evades heuristic detection.

**Mechanism:** Modern detection systems identify malware by comparing behavior against known patterns. A VRAL-based attacker would use the delta constraint of its transition function in reverse: each child capsid differs from its parent by less than the detection threshold, but the accumulation of deltas across generations produces effective code mutation without any individual step triggering an alert.

**Why the no-syscall constraint neutralizes it:** The Polymorphic needs to execute real mutations — modifying executable code in memory, rewriting binaries. Without syscalls, the mutations are arithmetic operations on internal payloads. They do not translate into observable actions outside the kernel. The malware "mutates" but the mutations have no effect on the system.

### Profile C — Offensive Consensus

**Hypothetical offensive capability:** Distributed coordination without a command-and-control server.

**Mechanism:** Traditional malware must contact a C2 server to receive instructions — a detectable and blockable communication pattern. VRAL capsids coordinate through topological neighborhood interference: when one finds a vulnerability or target, it propagates that finding to its neighbors. The swarm converges on the breach without any packet leaving toward an external server.

**Why the no-syscall constraint neutralizes it:** Offensive Consensus needs to act on the target it finds — exploit the vulnerable service, exfiltrate data, escalate. Without syscalls, the swarm convergence is mathematically precise but operationally null. Coordinated capsids cannot execute any action on the system.

### Profile D — The Grey Plague

**Hypothetical offensive capability:** Denial of service through resource exhaustion without detectable saturation.

**Mechanism:** The most subtle profile. The attacker does not aim to steal or infect anything. It disables VRAL's termination consensus and allows capsids to reproduce without limit. Because they use stack allocation, memory does not grow visibly. CPU usage rises from a hyper-simulation of inert computations, elevating processor temperature until the hardware degrades. The operating system sees what appears to be normal scientific computation.

**Why the no-syscall constraint partially neutralizes it:** The Grey Plague is the only profile with a real effect without syscalls — heat is physical and does not require OS access. The no-syscall constraint prevents the attacker from exfiltrating results or acting on the system, but does not prevent thermal damage. Defense against the Grey Plague requires complementary architectural controls in the orchestrator: a hard cap on capsid population, a thermodynamic watchdog, and mandatory termination consensus.

---

## 4. The orchestrator boundary

The orchestrator is responsible for everything the kernel cannot do:

- Read sensor data, network packets, or kernel events
- Serialize observations into flat, pointer-free data structures
- Write the serialized data into a pre-allocated memory buffer shared with the kernel
- Read the output buffer where the kernel writes verdicts
- Execute physical responses (block a port, raise an alarm, log an event) based on those verdicts

The orchestrator has syscalls. The orchestrator is attackable through classical software vulnerabilities. **But the orchestrator does not contain the detection logic** — that lives in VRAL. Compromising the orchestrator allows an attacker to blind or ignore the detector; it does not allow poisoning the capsids or capturing their model state.

---

## 5. The boundary principle

**Crossing VRAL → Orchestrator:** scalar verdicts (integers, flags). No pointers, no executable code, no control information about the orchestrator.

**Crossing Orchestrator → VRAL:** pre-processed observations in flat, immutable structures of the size and shape expected by the kernel. The kernel validates structural shape; it does not interpret content.

**Never crossing the boundary:** shared references, callbacks, OS handles, kernel capabilities, pointers to orchestrator memory.

This boundary separates the defensible (kernel with verifiable properties) from the attackable (orchestrator with a classical surface). Compromising the orchestrator is a conventional defense problem. Compromising the kernel is mathematically difficult.
