# cedarwood

Efficiently-updatable double-array trie in Rust, ported from C++ cedar.

## Build & Test

```bash
cargo build                              # debug build
cargo build --release                    # release build
cargo test                               # run all tests
cargo test --features reduced-trie       # test with reduced-trie feature
cargo bench                              # run criterion benchmarks
cargo doc --open                         # generate and view rustdoc
```

## Project Structure

```
src/lib.rs          -- entire library: data structures, Cedar impl, tests
benches/
  cedarwood_benchmark.rs   -- criterion micro-benchmarks
  macro-benchmark/         -- real-dictionary benchmark binary
  cpp/                     -- C++ cedar benchmark for comparison
docs/                      -- architecture and design documentation
```

## Code Conventions

- Single-file library (`src/lib.rs`). All types, methods, and tests live here.
- Internal structs (`NInfo`, `Node`, `Block`) are private; only `Cedar`, `PrefixIter`, and `PrefixPredictIter` are public.
- Feature flag `reduced-trie` enables a more compact trie representation via `#[cfg(feature = "reduced-trie")]` blocks throughout the code.
- Uses `smallvec` for stack-allocated child collection during conflict resolution (avoids heap allocation for up to 256 children).
- Widening casts use `i32::from(u8_val)` instead of `as i32` (clippy compliance).
- Edition 2021, formatted with `rustfmt` (config in `rustfmt.toml`).

## Key Implementation Notes

- The double-array stores `base` and `check` in `Node`. Negative values in `base`/`check` indicate free slots forming a doubly-linked free list.
- Blocks are 256-element chunks classified as Open (>1 free), Closed (1 free), or Full (0 free), managed as cyclic doubly-linked lists.
- Conflict resolution (`resolve`) relocates the smaller sibling set to minimize work.
- The `reject` heuristic prunes block searches: if a block's minimum rejection threshold exceeds the number of children needed, the block is skipped.
