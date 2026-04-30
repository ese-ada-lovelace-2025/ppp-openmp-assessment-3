# Extension: NUMA first-touch

Demonstrate that parallel first-touch initialisation places memory pages on the NUMA domain of the writing thread, avoiding cross-domain traffic during the stencil sweep.

## What to deliver

Two variants, both in this directory:

- `stencil_naive.cpp` — stencil with **single-threaded** initialisation of `u` / `u_next`. All pages end up on one NUMA domain; the 128-thread run pays cross-node bandwidth penalties.
- `stencil_ft.cpp` — identical stencil but with **parallel first-touch** initialisation: each thread writes the region it will later read.

Both compile to separate binaries. CI runs both at 128 threads on Rome, computes `delta_percent = (naive_time - ft_time) / naive_time × 100`, and expects a substantial positive delta (≥ 15% for full marks, ≥ 5% for half).

## Precondition: use `posix_memalign`, NOT `std::vector`

`std::vector<double>(N)` value-initialises every element at construction time — i.e. the constructing thread (master) writes zeros across the entire array, and that counts as the first touch. Pages are pinned to the master's NUMA domain before any parallel-init loop ever runs. **You will not see a delta if both variants use `std::vector`** — both are then effectively master-init.

Allocate uninitialised, aligned memory in both variants:

```cpp
const std::size_t bytes = NX * NY * NZ * sizeof(double);
void* a_raw = nullptr;
posix_memalign(&a_raw, 64, bytes);          // 64 = cache-line / SIMD alignment
auto* a = static_cast<double*>(a_raw);
// ... use a ...
std::free(a);
```

The `core/stencil.cpp` starter already uses this pattern; copy from there.

## Implementation hints

1. Copy `../core/stencil.cpp` (relative to this directory) as the starting point for both variants — it already uses `posix_memalign`.
2. In `stencil_naive.cpp`, replace the parallel init loop with a plain serial one.
3. In `stencil_ft.cpp`, keep `#pragma omp parallel for` over the init loops, with the **same iteration order** as the compute step. This is the whole point — traversal order at init must match the compute step to get correct page placement.
4. Both must produce the same checksum (correctness gates the delta).

## What the grader reads

- `perf-results-a3-ext.json` — CI-measured times at 128T for both variants.
- `EXTENSION.md` — your declared before/after times (CI cross-checks within 10%).

## Reading list

- Day 4 slide deck — "First-touch policy", "First-touch — the `std::vector` trap", "First-touch — `posix_memalign` + parallel init".
- `docs/rome-inventory.md` — numa-hardware output shows exactly which cores map to which NUMA domain.
- `snippets/day4/numa_first_touch.cpp` in the lectures repo — minimal working example using `posix_memalign`.
