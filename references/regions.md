# Instrumented regions

Use region mode to exclude setup and teardown from hardware counts. The target must call the matching API; `--region` alone does not select a function or source range.

## C and C++

Use `include/apmc.h` from the matching [upstream revision](https://github.com/0ax1/apmc/blob/9cff99a9df2ae055f1e3065c64f633e38bb6beec/include/apmc.h). A Cargo binary install does not install this header into a system include directory. Set `apmc_source` to an existing checkout before compiling:

```sh
apmc_source=/path/to/apmc
cc -O3 -I "$apmc_source/include" region_demo.c -o region_demo
```

Example `region_demo.c`:

```c
#include <apmc.h>
#include <stdio.h>

int main(void) {
    volatile unsigned long long result = 0;
    /* Prepare inputs before the marker. */
    apmc_start();
    for (unsigned long long i = 0; i < 1000000; ++i) {
        result += i;
    }
    apmc_stop();
    printf("%llu\n", result);
    return 0;
}
```

This loop is only an instrumentation smoke test; the volatile accumulator affects the work being measured. For a real benchmark, mark its actual kernel and preserve observable results without changing the algorithm. The same API names apply in C++; check that the target compiler accepts the upstream header's C atomic declarations.

The header resolves `apmc_start_impl` and `apmc_stop_impl` with `dlsym`. Merely declaring external `apmc_start()`/`apmc_stop()` functions does not provide the wrapper implementation.

## Rust

Use an existing compatible `apmc` dependency when present. For the probed version, this dependency pins the matching revision:

```toml
[dependencies]
apmc = { git = "https://github.com/0ax1/apmc", rev = "9cff99a9df2ae055f1e3065c64f633e38bb6beec" }
```

```rust
fn main() {
    let mut result = 0u64;
    apmc::region::start();
    for i in 0..1_000_000u64 {
        result = result.wrapping_add(std::hint::black_box(i));
    }
    apmc::region::stop();
    println!("{result}");
}
```

Build separately with `cargo build --release`, then measure the resulting executable. Keep any temporary instrumentation or dependency change within the requested benchmark work.

## Run and interpret

```sh
apmc_bin="$(command -v apmc)"
sudo -n "$apmc_bin" --no-color stat --region \
  -e L1D_CACHE_MISS_LD,BRANCH_MISPRED_NONSPEC -- ./region_demo
```

- Markers apply to the **calling thread**. Place balanced start/stop calls inside each worker whose work should count; wrapping a spawn/join call in the parent does not select the worker's counters.
- Multiple pairs accumulate into one total across participating threads and regions. There are no region labels or per-region output rows.
- Avoid nested starts. The implementation takes a new snapshot on each start, so an inner start overwrites the outer baseline. Stop on the same thread, including error paths; an unfinished region may extend into teardown collection.
- The wrappers do nothing without the injected library. With ordinary `stat` but without `--region`, the marker implementations also do nothing and automatic whole-program counting remains active.
- Run the instrumented executable normally first to check its result, then verify that region mode produces plausible nonzero fixed counters. An uninstrumented executable can produce zero region counts without a configuration error.
- The printed wall time remains whole-command time. Use a timer around the region if the task needs its runtime or throughput. Keep marker/timer overhead in mind for very short regions.

These semantics come from the matching `src/region.rs`, `include/apmc.h`, and `inject/kpc_inject.c`. See [probe-notes.md](probe-notes.md) for the validation boundary.
