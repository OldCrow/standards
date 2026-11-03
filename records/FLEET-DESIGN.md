# Fleet Design (libhmm, libstats, corvus, and the libraries after them)

Status: **PROPOSAL — drafted 2026-11-02.** This is a design record, not a
cross-repo effort in progress. The corvus adoption workplan remains the one
active record. Section 2 (the peer rule) is **decided [user, 2026-11-02]**.
Everything else is `[OPEN]`, and section 10 says which decision each
upcoming milestone needs, and when. Sources: a design discussion and a code
survey on 2026-10-26, covering libhmm v4.4.1, libstats v2.5.0 and corvus on
`dev/v1.2.0`.

## Why this exists

The fleet aims to be an **accuracy-first statistical computing layer**:
contractual accuracy bounds that hold on every ISA tier, double precision,
and CPU SIMD throughput. It sits between NumPy/SciPy (where accuracy is
good but not contractual) and PyTorch/JAX (GPU-first, float32 by default).
Bit-for-bit reproducibility *across* tiers is not part of the proposition:
libhmm disclaims it (its AGENTS.md, Architecture: tier dispatch changes the
width of its partial sums), and corvus does not promise it across versions.

New libraries are in view (libkalman, liboptim, libsde), and so is a GUI.
Each new library adds pins, bindings and an adoption round. Each shared
kernel left in more than one place drifts. *Incident: the same clean-room
trig kernel ended up in three copies (libstats #95 → libhmm #74 → two tables),
and the 2026-08-21 review fixed the same gate defects twice.* This record
sets the shape before the next library exists, so that adding one does not
deepen the dependency graph or copy code.

The model is Unix: small tools that each own one domain, a shared
interchange format, and composition above the tools rather than inside
them.

## 1. Where the fleet stands [DERIVED, 2026-10-26 survey]

| Concern | corvus | libstats | libhmm |
|---|---|---|---|
| Fleet dependencies | none | none (corvus from v3.0.0) | none (corvus via the #106 spike) |
| Public types | `std::span` only | own class hierarchy, `PerformanceHint` | own `BasicMatrix`, `Hmm` |
| Threads | none | own pools and backends (~2,900 lines) | none; caller-level parallelism is the contract (pool moved to `tools/` in `34ee3d0`) |
| Random generator | none | `std::mt19937&` in virtual `sample` | `std::mt19937_64&` in virtual `sample` |
| Aliasing of batch in/out | exact aliasing allowed | overlap forbidden | — |
| SIMD dispatch | internal (Highway) | own, heavy | own function table |
| Distributions | none | 27 | 16 univariate + 3 multivariate |
| Weighted fit | — | none | virtual `fit(span data, span weights)` |

The duplication that remains is distribution code (Newton MLEs for Gamma,
Beta, Student-t and Weibull in both libraries) and SIMD/elementary kernels.
The corvus adoptions remove the elementary and special-function kernels.

## 2. The peer rule — DECIDED [user, 2026-11-02]

**A fleet library depends on no other fleet library except a leaf.** Today
the only leaf is corvus. Peers (libstats, libhmm, and the libraries that
follow) never depend on each other. Optional bridges connect peers
(section 6), and consumers compose them.

- **It rules out:** libhmm depending on libstats. The #106 spike stays a
  corvus-only spike; libstats distributions reach libhmm through a bridge
  or not at all.
- **It allows:** external leaves (Highway under corvus; a linear-algebra
  library if one is chosen, section 7) and any number of peers above corvus.
- **Every repo meets it today**, so it is a rule with followers, not an
  aspiration.

Why: each peer-to-peer edge is a pin bump that propagates up the graph,
and a binding that inherits a second C++ dependency. A library two levels
deep is the most this fleet needs.

## 3. The library contract [OPEN]

The proposal is to take corvus's public contract as every fleet library's
target, the interchange format between them:

1. Batch entry points take spans in and spans out (`std::mdspan` for
   matrices, section 7).
2. The public headers include only the C++ standard library.
3. The library creates no threads (section 5).
4. Randomness comes in through a template parameter satisfying
   `std::uniform_random_bit_generator`, not a named engine.
5. Batch calls do not allocate on the hot path, and say so.
6. One aliasing rule, fleet-wide.

Open:
- **Aliasing.** corvus allows exact aliasing; libstats forbids any
  overlap, because several kernels re-read their input after writing
  (libstats #112). Settle on one rule, or let each library state its own
  in the same place.
- **Virtual APIs and random generators.** Both libstats and libhmm put
  the engine in a *virtual* `sample`, and virtual functions cannot be
  templates. The options are a non-virtual template layer over a virtual
  "uniform stream" hook, or type erasure, which costs one indirect call
  per draw (amortised by batch sampling).
- **Register-level kernels.** libhmm's distribution kernels call
  `exp_pd`/`log_pd` in-register. corvus's span-only contract cannot serve
  them, and corvus rules out a register-level export (corvus PLAN.md). So
  the contract binds the public API, not a library's internal kernels.

## 4. Layers [OPEN]

```text
Layer 0  leaves         corvus; [linalg]; [rng]; libpar (optional)
Layer 1  peers          libstats, libhmm, libkalman, libsde → Layer 0 only
                        liboptim → nothing (linalg for Newton/trust-region)
Layer 2  bridges        header-only adapters between two peers (section 6)
Layer 3  composition    py* bindings, GUI, an eventual libprob
```

libprob, if it exists, is a composition layer that forms late, once
several peers show duplication it would remove. It is not a foundation to
build first.

## 5. Parallelism [OPEN]

Libraries never create threads. They split work into **fixed chunks
whose boundaries depend only on the data size**, and they run those chunks
on an executor the caller supplies, serially by default. An optional
Layer 0 library, libpar, provides executors for callers who want one.

Why:
- **Threads add no nondeterminism.** With a fixed partition and an
  ordered combine, a parallel sum is bit-identical across thread counts
  and runs on one machine. Default TBB and OpenMP reductions are not.
  Across tiers, results still differ where the SIMD kernels differ;
  threading just adds nothing on top.
- **No oversubscription.** A parallel caller, such as multi-start fitting,
  calling a library with its own pool multiplies the thread count.
- **The standard library is not there yet.** Parallel algorithms in
  libc++ are incomplete, and C++26 `std::execution` is not usable across
  the fleet's compilers. Shaping the executor like a `std::execution`
  scheduler lets libpar shrink to an adapter later.
- **Python users need a default.** Bindings release the GIL and pass a
  libpar executor, so Python users keep today's batch speed after
  libstats stops threading internally.

libhmm already works this way. No threads are used in its production
path, and concurrent training of separate instances is a documented,
TSan-tested contract (`tests/test_concurrent_training.cpp`; libhmm
AGENTS.md, Architecture).

Evidence, from libstats:
- Batch pdf/logpdf/cdf already use fixed 1,024-element slices with
  disjoint outputs, so they are bit-reproducible today.
- PARALLEL and WORK_STEALING differ only in scheduling.
- The one live cross-chunk combine, Gaussian `fit`, uses a grain that
  varies by platform (libstats #283).
- The thread-count-dependent reduction helpers have no caller (libstats
  #284).
- The extraction moves about 2,900 lines and deletes about 1,000. The
  single seam is `autoDispatch` (libstats #282). The caller-supplied
  executor is libstats #285, gated on this section.

Open:
- Whether the fleet states the guarantee above (thread-count
  independence on a given machine and tier) as a contract, and in which
  repos.
- The executor concept's exact shape.
- Whether libpar exists, or bindings use TBB directly.
- Whether a generic executor scheduling fixed slices matches libstats'
  work-stealing throughput, which was measured faster on macOS. This needs
  a release-mode benchmark on every fleet machine.

## 6. Composition concepts and bridges [OPEN]

Three concepts carry most of the composition between peers.

**Emission.** Batch log-density, plus a weighted fit (for EM's M-step).
libhmm's inference core already builds a T×N log-emission matrix
(`fillLogEmissions`, consumed only through `logEmitByTime_`). So the
proposal is that libhmm's inference API takes that matrix as an
`mdspan`, and the Emission concept is needed only for training.

```cpp
// [ILLUSTRATIVE]
template <class E>
concept Emission = requires(const E& e, E& m, std::span<const double> x,
                            std::span<const double> w, std::span<double> out) {
    e.log_pdf(x, out);   // batch, no aliasing promise beyond section 3
    m.fit(x, w);         // weighted MLE
};
```

libstats has no weighted fit today, so an hmm+stats bridge needs it.
Weighted MLE is worth having in a statistics library anyway.

**Objective.** A value and a gradient over spans, plus bounds; liboptim
consumes it, and libkalman and the libstats MLEs produce it.

```cpp
// [ILLUSTRATIVE]
double f(std::span<const double> x, std::span<double> grad);
```

**Executor.** Section 5.

**Bridges** are header-only adapters between exactly two peers, and
optional. Candidates:
- hmm+stats: libstats distributions as libhmm emissions;
- kalman+optim: a likelihood as an objective;
- sde+stats: stationary laws as distributions;
- hmm+kalman: switching state-space models and IMM.

Open:
- Where bridges live: in the higher-numbered peer's repo, in each
  consumer, or in one small `fleet-bridges` repo that only end consumers
  depend on.
- How concepts are spelled (concepts inside a library; type erasure such
  as `std::function_ref` at coarse boundaries).

## 7. Shared vocabulary [OPEN]

- **Matrices.** `std::mdspan` is C++23, and the fleet is C++20. corvus's
  VERSIONING.md makes a standard bump a major change. The options are the
  Kokkos reference implementation as a header-only fallback (an external
  leaf) or a C++23 bump at each library's next major version. Check
  per-compiler support with the cross-platform toolchain ledger before
  choosing.
- **Linear algebra.** libkalman needs dense Cholesky and QR, as do
  liboptim's Newton and trust-region methods. Eigen as an external leaf
  fits the peer rule; a small in-house set fits accuracy-first control.
  libhmm already has a Cholesky (`linalg/cholesky.h`). The first library
  that needs it decides.
- **Random generator.** See section 3. Unify the engine *type* too:
  libstats uses `mt19937` (32-bit), libhmm `mt19937_64`.
- **No fleet "core types" library.** Rejected: it becomes the hub every
  library pins, which is the stacking this record avoids. The C++
  standard is the only free shared vocabulary.

## 8. Candidate libraries [OPEN]

Tentative order after the current adoptions [user, 2026-10-26]: libkalman,
then liboptim, then libsde. A GUI comes after libkalman, if at all. libprob
comes late, as a composition layer.

- **libkalman** is the continuous-state sibling of libhmm (RTS smoothing
  ↔ forward-backward, Kalman EM ↔ Baum-Welch).
  - Square-root forms (Bierman-Thornton UD, Potter) match the
    accuracy-first doctrine.
  - IMM and switching models are the natural hmm+kalman bridge.
  - Particle filters are a different discipline (Monte Carlo, resampling,
    RNG quality); later, or separate.
  - v1 should be EM-only, so it needs no optimiser.
- **liboptim** is *new capability*, not an extraction. The survey found
  no shared minimiser code to pull out. The fleet's optimiser-like code
  is either distribution-specific Newton steps or special-function
  inverses that corvus absorbs.
  - Scope: statistical estimation. Bounded and transformed parameters,
    L-BFGS-B, Newton/Fisher scoring, trust region, standard errors, and
    multi-start with population batching.
  - Its first real consumer is libkalman's gradient-based fitting.
  - It depends on nothing.
- **libsde** is the weakest fit. Euler–Maruyama and Milstein are common
  everywhere; exact simulation (OU, CIR via noncentral χ²) fits the
  doctrine but needs corvus specials that do not exist (noncentral χ²,
  Bessel K). It is also reachable through continuous-discrete Kalman
  filtering.
- **GUI.** Qt, single framework (ewcalc proves the toolchain). It is worth
  building once it can show both discrete and continuous latent models.
  Its value is teaching and demonstration. If it is meant to draw outside
  users, it is a product with its own long-term cost.

## 9. Rejected alternatives

- **libhmm adopts libstats as a dependency.** Ruled out by section 2;
  replaced by the hmm+stats bridge.
- **A libstats2 beside libstats.** It would duplicate the distributions'
  numerics, with every fix made twice. The alternative is a major version
  with deprecations in the preceding minor releases, and optionally a
  versioned inline namespace. Revisit only if the new model replaces
  libstats' public class structure wholesale.
- **liboptim built by extraction.** There is nothing to extract (section 8).
- **corvus exports its dispatch or register-level kernels.** corvus's
  docs rule it out. It would make corvus a framework rather than a
  function library.
- **Selectable fast/accurate modes.** Ruled out by corvus's contract
  (fleet review 2026-08-21); speed is a property of the ISA tier.

## 10. What each milestone needs from this record

| Milestone | Needs | Status |
|---|---|---|
| libstats v3.0.0 (corvus adoption) | nothing beyond corvus as leaf | ready |
| libhmm #106 spike | section 2 | decided |
| libstats #282 (single dispatch seam) | nothing; internal | ready |
| libstats #285 (caller executor) | section 5: executor concept | [OPEN] |
| libhmm v5.0.0 (#95 `train()` split) | section 6 Emission; section 7 matrices and RNG | [OPEN]; settle before v5.0.0 planning |
| libstats next major (post-v3 refactor) | sections 3, 5, 7; weighted fit for section 6 | [OPEN]; settle before #4's scope is fixed |
| libkalman start | section 7 linear algebra; sections 5 and 6 | [OPEN] |
| liboptim start | section 6 Objective | [OPEN] |

Next design step: specify section 3 and the three concepts in section 6,
then answer the libhmm v5.0.0 row first, because it is the earliest
milestone that changes a public API.
