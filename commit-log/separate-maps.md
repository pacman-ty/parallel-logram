# Parallelizing dictionary building with per-thread HashMaps and merge

## Summary

Added dictionary construction path `--single-map` that splits the log file into equal chunks. Each chunk gets a separate thread with its own local `HashMap` for both the 2-gram and 3-gram dictionaries. It merges the per-thread maps by summing counts after all threads complete. This is activated by passing `--single-map` on the command line and and can be modified with the `--num-threads` argument.

## Tech details

The bottleneck in the starter code is `dictionary_builder` in `parser.rs`.

To parallelize this:

1. Entire log file is loaded into memory as a `Vec<String>` via `read_all_lines` method
2. Divided into `num_threads` contiguous chunks
3. Each thread processes its chunk of lines sequentially, 

Within each thread:

- build `dbl` and `trpl` HashMaps
- computes the `prev1`/`prev2` tokens by tokenizing that boundary line via `get_initial_prev`
- use the next line in the file as its lookahead, even if that line belongs to the next thread's chunk 

This perserves double counter behaviour. 

Threads are created using `std::thread::scope`. This allows the closure to borrow the shared `Vec<String>` and `Vec<Regex>` directly without needing `Arc` wrappers.

Once all threads finish merge the result. For the dictionaries each key's counts are summed across all thread-local maps via `*final_dbl.entry(k).or_insert(0) += v`. The `all_token_list` vectors are merged in thread order, maintaining deduplication check `contains`.

## Testing for correctness

All seven existing unit tests pass (`cargo test`). 

Compared the sequential output `--num-threads 1` against the separate-maps output `--single-map --num-threads 8`. 

In all cases, the double dictionary length, triple dictionary length, all tokens count, and final dynamic tokens list were identical

Edge cases --num-threads 0 and --num-threads 1 both produce correct results

## Testing for performance

Benchmarks using hyperfine show clear speedups over the sequential baseline.

Command:

hyperfine 'cargo run --release -- --raw-hpc data/HPC.log --to-parse "58717 2185 boot_cmd new 1076865186 1 Targeting domains:node-D1 and nodes:node-[40-63] child of command 2176" --before-line "58728 2187 boot_cmd new 1076865197 1 Targeting domains:node-D2 and nodes:node-[72-95] child of command 2177" --after-line "58707 2184 boot_cmd new 1076865175 1 Targeting domains:node-D0 and nodes:node-[0-7] child of command 2175" --cutoff 106 --single-map --num-threads=x'


My results: (may vary machine to machine)

Benchmark 1: `--num-threads=1`

Time (mean ± σ): 4.602 s ± 0.029 s [User: 4.566 s, System: 0.031 s]

Range (min … max): 4.563 s … 4.654 s 10 runs

Benchmark 2: `--single-map --num-threads=8` (default)

Time (mean ± σ): 912.0 ms ± 17.9 ms [User: 3032.7 ms, System: 52.8 ms]

Range (min … max): 897.3 ms … 959.7 ms 10 runs
