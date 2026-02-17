# Parallelizing dictionary building with shared concurrent DashMap

## Summary

Added a concurrent dictionary construction path (the default when `--single-map` is not specified) that uses `DashMap` as a concurrent replacement for `HashMap`. All threads share a single `DashMap<String, i32>` for the 2-gram dictionary and one for the 3-gram dictionary. They atomically increment counts without requiring a post-join merge step. This is the default parallel mode.


## Tech details

The concurrent implementation follows the same chunking strategy as the separate-maps approach: 

1. Entire log file is loaded into a `Vec<String>`
2. Split into `num_threads` contiguous chunks
3. Threads processes chunks sequentially 

The key difference is that instead of thread-local `HashMap`'s, all threads write to two shared `DashMap` instances. 

`DashMap` internally partitions keys across multiple shards, each protected by its own `RwLock`. This way inserting different keys rarely contend. 

Insertions use `*dbl.entry(key).or_default() += 1`, which acquires a per-shard write lock only for the duration of the single increment operation.

A separate function `process_dictionary_builder_line_concurrent` accepts `&DashMap<String, i32>` references instead of `&mut HashMap<String, i32>`. Otherwise the function is identical to `process_dictionary_builder_line`. It perserves the exact n-gram generation logic including the double-count at line boundaries. 

`std::thread::scope` is used so the `DashMap` references can be borrowed directly by each thread without `Arc`. The `all_token_list` is still collected per-thread (into local `Vec<String>`s) and merged after joining. This is due to the fact it is only used for its length in the output and does not need concurrent access.

After all threads complete `DashMap` instances are consumed via `into_iter()` and collected into standard `HashMap`s for compatibility with the existing `print_dict` and parsing code downstream. 

The `dashmap` crate was chosen because it is actively maintained and has API similar to `std::collections::HashMap`

## Testing for correctness

All seven unit tests pass (`cargo test`)

I compared the output of the concurrent-maps path (`--num-threads 8`) against the sequential path (`--num-threads 1`)

In all cases, the double dictionary length, triple dictionary length, all tokens count, and final dynamic tokens list were identical 

Edge cases `--num-threads 0` and `--num-threads 1` both produce correct results

## Testing for performance

Benchmarks using `hyperfine` show clear speedups over the sequential baseline. 

<br>

Command:
```
hyperfine 'cargo run --release -- --raw-hpc data/HPC.log --to-parse "58717 2185 boot_cmd new 1076865186 1 Targeting domains:node-D1 and nodes:node-[40-63] child of command 2176" --before-line "58728 2187 boot_cmd new 1076865197 1 Targeting domains:node-D2 and nodes:node-[72-95] child of command 2177" --after-line "58707 2184 boot_cmd new 1076865175 1 Targeting domains:node-D0 and nodes:node-[0-7] child of command 2175" --cutoff 106 --num-threads=x'
```

<br>

**My results:** (may vary machine to machine)

**Benchmark 1:** --num-threads = 1

Time (mean ± σ): 4.602 s ± 0.029 s [User: 4.566 s, System: 0.031 s]

Range (min … max): 4.563 s … 4.654 s 10 runs

**Benchmark 2:** --num-threads = 8 (default)

Time (mean ± σ):     789.7 ms ±  14.1 ms    [User: 3190.1 ms, System: 56.1 ms]

Range (min … max):   771.5 ms … 809.5 ms    10 runs


