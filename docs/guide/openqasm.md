---
title: OpenQASM support
description: The OpenQASM 2.0 and 3.x language subset accepted by QPUTwin's from-scratch parser — includes, gates, modifiers, measurement, reset, classical control and error messages.
---

# OpenQASM support

QPUTwin's front-end (`qputwin_core.qasm`) is a hand-written regex lexer, a recursive-descent parser and a flattener, all pure Python with no parser library. It never calls `eval`, and `include` never touches the filesystem.

## OpenQASM 2.0: complete

* `OPENQASM 2.0;`, `include "qelib1.inc";`
* `qreg`, `creg`
* `gate name(params) args { body }`
* `U(θ,φ,λ)`, `CX`, and every `qelib1.inc` gate
* `measure a -> b;`, `reset a;`, `barrier …;`
* `if (creg == int) qop;`
* register broadcasting: `h q;`, `cx q, r;` (equal sizes), `measure q -> c;`

`opaque` gates are rejected (`UnsupportedFeatureError`), because an opaque gate has no defined unitary.

## OpenQASM 3.x: practical subset

| Feature | Supported |
|---|---|
| `OPENQASM 3;` / `3.0;`, `include "stdgates.inc";` | ✓ |
| `qubit[n] q;`, `qubit q;`, `bit[n] c;`, `bit c;`, legacy `qreg`/`creg` | ✓ |
| hardware qubits `$n` | ✓ |
| `gate` definitions, builtins `U`, `gphase`, `CX` | ✓ |
| `c = measure q;`, `c[i] = measure q[i];`, `measure q -> c;` | ✓ |
| `reset`, `barrier`, `delay[...]` (accepted; an ideal twin has no free evolution outside the schedule) | ✓ |
| modifiers `inv @`, `pow(k) @` (integer k), `ctrl(n) @`, `negctrl(n) @` on any gate | ✓ |
| `if (c == n) { … }`, `if (c != n)`, `if (c[i] == b)` | ✓ |
| loops, subroutines (`def`), classical arithmetic and types, `extern`, `defcal`/pulse, arrays | ✗ (`UnsupportedFeatureError` with the line number) |

## Libraries and vendor natives

* `qelib1.inc` is embedded verbatim, and `stdgates.inc` is written in terms of `U`, `CX` and `gphase`.
* Vendor-native gates are available on every backend: `u1q`, `zzmax`, `rzz`, `gpi`, `gpi2`, `ms`, `zz`, `prx`, `rx`, `cz` and `iswap`. A circuit written in one vendor's natives is therefore portable to another vendor, where it is compiled.
* Gate bodies are **lexically bound**: redefining `h` cannot change the library's `ccx`.
* `ctrl @` is exact. A single control uses the ABC decomposition; more controls use Barenco et al.'s Lemma 7.9 recursion with principal square roots.

!!! note "Global phase in QASM 2 `rzz`"
    On Helios `rzz` is native and kept opaque with the canonical matrix $e^{-i\theta/2\,Z\otimes Z}$. `qelib1.inc` defines `rzz` as `cx; u1(θ); cx`, which equals that matrix times $e^{i\theta/2}$. The two differ only by a global phase, which matters only when comparing exact statevectors across devices for QASM 2 circuits. Write such circuits in OpenQASM 3 (`stdgates.inc` is phase-exact) when you compare phases.

## Physical qubits and registers

Registers are laid out in declaration order: `qreg a[2]; qreg b[3];` gives `a[0]→0 … b[2]→4`. These indices are **physical** qubits; without [routing](routing.md), no layout pass moves them. Bit strings print as `c[n-1]…c[0]` (little-endian, as in the OpenQASM convention).

## Error messages

| Error | Example |
|---|---|
| `QASMSyntaxError` | `expected ';' (line 3, col 9)`, with `details.line` and `details.col` |
| `QASMSemanticError` | undeclared register, index out of range, arity mismatch, unknown include |
| `UnsupportedFeatureError` | `for` loop on line 5 |

Every operation keeps its `origin` (the original statement and line), so errors raised later, such as topology violations, quote what *you* wrote rather than an internal decomposition.
