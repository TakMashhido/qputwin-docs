---
title: Simulation engines
description: QPUTwin's statevector, density-matrix and quantum-trajectory engines — when each is used, how they work, precision and qubit-width limits.
---

# Simulation engines

| Engine | State | Exact? | Width in a 12 GiB worker | Used for |
|---|---|---|---|---|
| `statevector` | $2^n$ amplitudes | exact (ideal) | 28 qubits (29 at complex64) | ideal runs |
| `density_matrix` | $4^n$ entries | exact (noisy) | 14 qubits | noisy runs while $2^n \le 4T$ and $n \le$ `QPUTWIN_DENSITY_AUTO_MAX_QUBITS` (12) |
| `trajectory` | $2^n$ per trajectory | exact when trajectories = shots | 28 qubits | wide noisy runs, leakage, quasi-static detuning |

## Automatic selection (`engine: "auto"`)

![Engine selection](../assets/diagrams/engine_selection.svg){ .qt-diagram }

For a noisy run, `auto` follows a measured cost model. A fused density-matrix step costs about $4^n$ and a batched trajectory step about $T\cdot 2^n$, where $T$ is the trajectory count (`noise.trajectories`, by default one per shot, at most `QPUTWIN_MAX_TRAJECTORIES`). The density matrix is chosen while $2^n \le 4T$, it fits in memory, and $n \le$ `QPUTWIN_DENSITY_AUTO_MAX_QUBITS` (12). It is also kept whenever the trajectories would not be one per shot, because it is exact for any shot count. For 1,024 shots this gives the density matrix up to 12 simulated qubits; for 64 shots, up to 8.

Override it with `options.engine`. Asking for a combination that cannot work, such as noise on `statevector` or leakage on `density_matrix`, returns a `400` error that explains why.

## Fused execution

Both exact engines run a branch-free program (every measurement terminal; the common case) through **gate fusion**:

1. **Plan.** The native gate list is fused into dense blocks of at most 5 qubits (statevector) or 2 qubits = 4 row/column axes (density matrix). The planner keeps several open blocks on disjoint qubits (they commute), grows a block while it fits, and carries small blocks into the next one rather than flushing them. Blocks made only of diagonal gates stay diagonal, are stored as their diagonal, and may span up to 12 qubits (statevector) or 12 axes (density matrix). Consecutive diagonal blocks are merged, because diagonals always commute. Matrix entries below 10⁻¹⁵ (rounding noise such as cos π/2) are set to zero; otherwise they decay, block after block, into denormal numbers that the CPU processes about 100× more slowly.
2. **Lazy activation.** The state starts as a single amplitude. A qubit that no block has touched yet is exactly `|0⟩` and is not stored. A block that reaches new qubits uses only the matrix columns where they are 0, so the stored state grows as the circuit spreads: a GHZ chain reaches full size only in its last block, and diagonal blocks never enlarge the state.
3. **Execute.** The state is a tensor whose axis order changes lazily. A block whose qubits are adjacent in that order is one batched `U @ X` with no data movement. Otherwise the block's qubits are gathered next to the lowest contiguous qubits, chunk by cache-sized chunk, and multiplied in the same pass. Diagonal blocks multiply by a phase table, also per cache chunk. The passes are split across `QPUTWIN_ENGINE_THREADS` threads (default: the worker's CPU quota).
4. **Read out without reordering.** Terminal measurements are sampled from the marginal in the stored order, and only the drawn outcomes are relabelled. The full state is never transposed back to standard order, and the probabilities are written into the state's own memory. Large distributions are sampled with a two-level inverse CDF: block totals first, then only the blocks a draw lands in.
5. **Trailing phases.** Diagonal gates after which no non-diagonal gate touches their qubits are dropped when no statevector is requested; they cannot change any measurement probability.

In the density-matrix engine every gate ($U\otimes U^*$), noise channel, depolarisation and reset is a superoperator on the (row, column) axes of its qubits, so **a gate and the noise after it become one matrix product**. The fused results equal the per-gate reference kernels to < 1e-12 (tested for every device, block width and thread count).

## Statevector

A dense vector in little-endian order: bit *q* of the index is simulated qubit *q*.

* Branch-free circuits use fused execution (above).
* The per-gate reference kernels update the state **in place** on reshaped views (peak ≈ 2× the state). They serve the shot-branching and trajectory paths.
* Diagonal gates (`rz`, `cz`, `rzz`, `zzmax`) are element-wise phase multiplications.
* Sparse two-qubit matrices skip their zero blocks.
* Terminal measurements are sampled once by exact inverse-CDF sampling, using O(shots) memory.
* Mid-circuit measurement, reset and feed-forward use **shot branching**: one simulation per reachable classical branch.

## Density matrix

ρ is stored as a $4^n$ vector, with row qubit q at bit n+q and column qubit q at bit q. Branch-free programs run as fused superoperator blocks (above); circuits with mid-circuit measurement use the per-gate kernels:

* unitary $U\rho U^\dagger$: $U$ on the row qubits, $U^*$ on the column qubits;
* one-qubit channel: one 4×4 superoperator $S = \sum_k K_k\otimes K_k^*$ on the (row, column) pair;
* depolarising: sliced partial trace;
* reset: a channel, not a branch;
* measurement probabilities: the real diagonal.

It reports `purity` = Tr ρ². With `return_density_matrix: true` (≤ 8 qubits, terminal circuits) you get ρ itself.

## Quantum trajectories

Each trajectory evolves a statevector and unravels each channel stochastically:

* Kraus operators are sampled with Born weights $p_k = \mathrm{Tr}(K_k\rho_qK_k^\dagger)$, computed from the qubit's reduced density matrix;
* depolarising becomes random Pauli jumps;
* quasi-static detuning is drawn once per trajectory;
* leakage is tracked per trajectory.

### Batched, fused trajectories

![Batched trajectory executor](../assets/diagrams/batched_trajectories.svg){ .qt-diagram }

A branch-free program without leakage, resets or quasi-static detuning (the common case) runs on the **batched executor**. It is exact in the same sense as the walk below, and it agrees with the exact density matrix in χ² tests at 500,000 trajectories.

1. **No-jump branch without the state.** For each Kraus channel, take the operator $K^\*$ whose $K^{\*\dagger}K^\*$ has the largest smallest eigenvalue $\alpha$. Its Born weight is at least $\alpha$ for every state. A uniform draw $u<\alpha$ therefore selects $K^\*$ without looking at the state, and only the rare draws $u\ge\alpha$ need the qubit's reduced density matrix. Depolarising jumps never depend on the state.
2. **Every draw up front.** A sub-batch of trajectories draws all its random numbers before any work, so it knows in advance which operations jump.
3. **One state for many trajectories.** About $2^{18}$ amplitudes' worth of trajectories share one $(n+r)$-qubit state: $r$ extra axes index the trajectory, and only per-row jump operations ever act on them. Rows stay unnormalised; each Born weight is taken relative to the row's own norm, and rows are rescaled only before their norm could underflow.
4. **Precomputed fused windows.** The gates, together with the no-jump branch $K^\*$ of every channel, are fused once per job into blocks for aligned windows of 96, 24 and 6 operations. A window without a jump runs its precomputed blocks on the fused executor. A window with a jump falls back to the next smaller window, and only the smallest window around a jump runs operation by operation. There, one-qubit operators wait as a pending 2×2 matrix until a two-qubit gate absorbs them.
5. **Jumps touch only their rows.** A jump works on strided views of its own rows in the executor's current axis order, so the batch is never reordered.

Sub-batches run concurrently, at most `QPUTWIN_TRAJECTORY_WORKERS` (default 4) at a time; the Python between NumPy calls holds the interpreter lock, and more than four was measured slower on a 16-core host. Each sub-batch has its own random stream, so results do not depend on the thread count. From $2^{20}$ amplitudes per trajectory, trajectories run one at a time, each on the multi-threaded fused executor. On the IBM Heron twin, a noisy QAOA-12 job (15 simulated qubits, 1,024 shots) runs in 3.3–3.4 s on four threads of the 16-core server. The diagnostics report `batched` and `batch_rows`.

### Sequential walk

Leakage, resets, mid-circuit measurement, feed-forward and quasi-static detuning keep the sequential walk, one trajectory at a time. Measurements and resets keep exact binomial shot branching.

* With **trajectories = shots** (the default, up to `QPUTWIN_MAX_TRAJECTORIES` = 2000), sampling is exact, and `result.noise.exact_sampling` is `true`.
* With fewer, the run samples the empirical mixture of $T$ trajectories; the error falls roughly as $T^{-1/2}$ until it reaches the shot-noise floor (see [benchmark](../research/benchmarks.md#trajectory-convergence)).

## Precision

`options.precision` (or `QPUTWIN_PRECISION`) selects `complex128` (the default) or `complex64`. Single precision halves memory. At 24 qubits it runs 1.7× faster on deep circuits on one thread (1.66× QV-style, 1.73× QFT adder) ([simulator comparison](../research/simulator-comparison.md#3-ideal-statevector-simulation)) and 1.2× faster on the GHZ chain of the [scaling benchmark](../research/benchmarks.md#engine-scaling), where little of the state is touched, and agrees with complex128 to within 1e-5 per amplitude, which is the tolerance the test suite checks.

## Memory and width limits

Two limits apply:

1. `QPUTWIN_MAX_SIM_QUBITS` caps the simulated width per worker. It defaults to the device size.
2. The **memory-feasibility guard** estimates the peak working set as 2 × state + 16 B per basis state + one state per pending branch (for a branch-free statevector run, which frees its spare buffer and writes the probabilities into the state's own memory, 2.25 × state + 128 MiB; at 31 qubits in complex64, 32.2 GiB measured against 36.1 GiB estimated), and compares it with the memory actually available to the container (cgroup v2 limit minus usage, or `MemAvailable`, minus a 256 MiB reserve). When two state buffers do not fit but one does, the fused executor switches to its **single-buffer (in-place) mode**: blocks are applied by gather, multiply and scatter back into the same buffer, and only an axis reordering briefly needs a second buffer. The peak is then 1.5 × state (the state plus half a state for the probabilities), at 5–17 % lower speed. `QPUTWIN_ENGINE_IN_PLACE` = `auto` (default), `always` or `never`; `result.workload.in_place` reports the choice.

A job that does not fit returns `422 ResourceLimitError` with `required_bytes`, `available_bytes` and `max_feasible_qubits`, so the worker is never OOM-killed. Only qubits a circuit **touches** count toward the width (active-qubit compaction).
