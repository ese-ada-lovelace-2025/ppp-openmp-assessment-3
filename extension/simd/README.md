# Extension: SIMD on the innermost loop

Demonstrate that `#pragma omp simd` produces measurable vectorisation gains on the innermost stencil loop.

## What to deliver

Two variants, both in this directory:

- `stencil_scalar.cpp` — stencil compiled **without** the `#pragma omp simd` annotation (rely only on autovectorisation).
- `stencil_simd.cpp` — stencil with `#pragma omp simd` on the innermost (k) loop.

Both compile to separate binaries.

## Precondition: aligned allocation

Both variants should allocate the stencil arrays with `posix_memalign(p, 64, n*sizeof(double))`. 64-byte alignment is the universal answer on Rome — it covers AVX2's 32-byte vector loads, aligns to cache lines, and doubles as a false-sharing-prevention size. Default `std::vector<double>(N)` gives you only `alignof(double) = 8` *and* value-initialises on the master thread, so it's wrong for both alignment and first-touch reasons.

The `core/stencil.cpp` starter already uses this pattern; copy from there as the starting point for both variants.

## Why the before/after may be small

Modern compilers (Clang 18, GCC 13) autovectorise simple stencils well without pragmas. The `simd` pragma typically helps by:

- Documenting independence: tells the compiler explicitly that the inner-loop iterations have no cross-iteration dependence, which it might not be able to prove for a more complex stencil.
- Documenting intent (helps compilers that are conservative about aliasing).

Your job is to *measure* the difference and *explain* it — a modest delta with a clear analysis (Bloom: Analyze) outscores a large unexplained one.

## Scoring specifics

- Full marks (5/5): `before/after ≥ 1.2×` (i.e. ≥ 20% speedup from SIMD).
- Half marks: `≥ 1.05×`.
- Attempt credit: below that, with the result grounded in your measurements.

## What the grader reads

- `perf-results-a3-ext.json` — CI times for both variants at 128 threads on Rome.
- `EXTENSION.md` — your declared times + ratio (reported as `delta_percent` = `(before - after) / before × 100`, so a 1.2× speedup corresponds to ~16.7%).

## Reading list

- Day 4 slide deck — SIMD section.
- `snippets/day4/simd_axpy.cpp` in the lectures repo — minimal `#pragma omp simd` example.
