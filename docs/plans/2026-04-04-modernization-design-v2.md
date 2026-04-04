# Cedarwood Modernization Design

**Date**: 2026-04-04
**Version**: 0.4.6 -> 0.5.0 (semver major bump due to semantic change)
**Revision**: v2

## Changes from previous version

This revision addresses all critical issues and incorporates suggestions from the [first design review](modernization-design-review-1.md).

### Critical issues addressed

1. **`Option<Vec>` return type (Critical #1)**: The reviewer recommended changing the return type of `common_prefix_search` / `common_prefix_predict` from `Option<Vec<...>>` to `Vec<...>`. After consideration, the user explicitly chose to **keep `Option<Vec<...>>`** and return `None` for empty results. This is a deliberate design decision: it distinguishes "no matches found" (`None`) from potential future use cases where the distinction matters (e.g., "prefix not in trie at all" vs "prefix exists but has no completions"), and it preserves API signature compatibility for callers already pattern-matching on `Option` or using `.unwrap()`. The reviewer's point about `Some(vec![])` being a confusing state is acknowledged -- under the new semantics, `Some(vec![])` will never be returned, eliminating the ambiguity. This section now documents the rationale explicitly.

2. **Benchmark file compilation (Critical #2)**: Added Section 6 covering the modernization of `benches/cedarwood_benchmark.rs`. The file uses `#[macro_use] extern crate criterion;` and `extern crate cedarwood;` which must be updated for edition 2021 and criterion 0.5 compatibility.

3. **`CEDAR_VALUE_LIMIT` dead_code (Critical #3)**: Changed approach from "remove `#[allow(dead_code)]`" to cfg-gating the constant itself: `#[cfg(feature = "reduced-trie")] const CEDAR_VALUE_LIMIT: i32 = i32::MAX - 1;`. This is cleaner than keeping the allow attribute, since the constant is only used inside `#[cfg(feature = "reduced-trie")]` blocks.

4. **`build()` dead_code annotation (Critical #4)**: Added explicit cleanup item. The `#[allow(dead_code)]` on `pub fn build()` (line 285) is unnecessary since public methods on public types never trigger dead_code warnings. This annotation will be removed.

### Suggestions incorporated

- **A (cast cleanup)**: Acknowledged as a separate commit. There are approximately 52 `as i32` casts, many of which are `u8 as i32` widening casts that clippy would flag. Each becomes `i32::from(...)`. This is a large mechanical change and will be done as its own commit to keep the diff reviewable.
- **B (pop_sibling annotation reasoning)**: Added a note explaining why the original author likely added `#[allow(dead_code)]` to `pop_sibling` (probable false positive from an older compiler version or leftover from a previous refactor where the caller was temporarily removed).
- **C (CI workflow modernization)**: Included as an optional bonus step (Section 9). The `.github/workflows/CI.yml` uses archived `actions-rs/*` actions and outdated action versions.
- **D (.travis.yml deletion)**: Added to Section 1. The file is dead and should be deleted, not merely un-excluded.
- **E (macro-benchmark check)**: Added to Section 6. The `benches/macro-benchmark/` sub-crate must be checked for compatibility (its `Cargo.toml` uses edition 2018 and a path dependency on cedarwood).
- **F (README example bug)**: Documented the existing bug in Section 5. The README examples call `.iter()` on `Option<Vec<...>>`, which iterates over 0-or-1 references to the inner `Vec`, not over the vector elements. The chained `.map(|x| x.0)` operates on `&Vec<(i32, usize)>`, not on `(i32, usize)`. This will be fixed.
- **G (unused_assignments detail)**: Expanded Section 4 with specific analysis of all 4 sites.

---

## Goal

Modernize the cedarwood crate to use current Rust idioms, fix potential bugs, eliminate unsafe code, and refresh project metadata. The library's behavior and public API signatures remain unchanged, but the semantics of `Option` return values are corrected.

## 1. Cargo.toml, Metadata & File Cleanup

- **Edition**: `2018` -> `2021`
- **Version**: `0.4.6` -> `0.5.0`
- **Dependencies**:
  - `smallvec`: `"1.6.1"` -> `"1.13"` (latest 1.x minor)
  - `criterion` (dev): `"0.4.0"` -> `"0.5"` (dev-only, minor API changes)
  - `rand` (dev): `"0.8.4"` -> `"0.8.5"` (latest patch)
- **Remove**: `[badges]` section (dead Travis CI / codecov entries)
- **Remove**: `/.travis.yml` from `exclude` list
- **Delete**: `.travis.yml` file from the repository root (dead CI config; CI now runs on GitHub Actions)

## 2. Unsafe Code Elimination

Rewrite `pop_sibling` from raw pointer (`*mut u8`) traversal to safe index-based traversal using a boolean `is_child` flag to track whether we're reading/writing the `child` or `sibling` field. Same logic, zero `unsafe`.

Remove `#[allow(dead_code)]` from `pop_sibling` since it is called unconditionally from `erase__` (line 455). The original annotation was likely a false positive from an older compiler version, or a leftover from a previous refactor where the calling code was temporarily removed.

## 3. Return Type Semantics & API Fixes

- **`common_prefix_search` / `common_prefix_predict`**: Keep `Option<Vec<(i32, usize)>>` signature but return `None` for empty results instead of `Some(vec![])`. This is a semantic breaking change (hence semver major bump).

  **Rationale for keeping `Option<Vec>`**: This was an explicit design choice. Returning `None` for "no matches" (rather than changing the signature to `Vec` directly) serves several purposes:
  - It distinguishes "no matches found" from potential future use cases where a richer distinction is needed (e.g., "prefix not in trie at all" vs "prefix exists but has no completions" for `common_prefix_predict`).
  - It preserves API signature compatibility for callers already pattern-matching on `Option` or using `.unwrap()`.
  - Under the new semantics, `Some(vec![])` will never be returned, so the `Option` wrapper has a clear meaning: `None` = empty, `Some(vec)` = non-empty results.

- **`Default` for `Cedar`**: Add a proper `Default` impl delegating to `Cedar::new()`, remove `#[allow(clippy::new_without_default)]`.

- **Constants**: Replace `std::i32::MAX` with `i32::MAX`.

- **`CEDAR_VALUE_LIMIT`**: Gate the constant with `#[cfg(feature = "reduced-trie")]` instead of using `#[allow(dead_code)]`. The constant is only used inside `#[cfg(feature = "reduced-trie")]` blocks (lines 309, 328, 720, 1060), so it is genuinely dead code in the default build. The cfg-gated approach is cleaner:
  ```rust
  #[cfg(feature = "reduced-trie")]
  const CEDAR_VALUE_LIMIT: i32 = i32::MAX - 1;
  ```

- **`build()` dead_code annotation**: Remove `#[allow(dead_code)]` from `pub fn build()` (line 285). Public methods on public types never trigger dead_code warnings, so this annotation is unnecessary.

- **Cast lint**: Remove `#[allow(clippy::cast_lossless)]` from impl block, fix individual sites where clippy flags them by converting `x as i32` to `i32::from(x)` for widening casts (e.g., `u8 as i32`). There are approximately 52 `as i32` casts in the file. **This will be done as a separate commit** to keep the diff reviewable. Only the widening casts that clippy actually flags need conversion; narrowing or same-width casts stay as-is.

- **Unused assignments**: Restructure variable declarations to eliminate `#[allow(unused_assignments)]` at the following 4 sites:

  1. **`follow()` line 346**: `let mut to = 0;` is immediately overwritten in both branches of the `if/else`. Fix: use `let to = if ... { ... } else { ... };` to assign directly from the conditional expression.

  2. **`find()` line 371**: `let mut to: usize = 0;` is overwritten on every iteration of the `while` loop at line 384. Fix: declare `to` inside the loop body where it is assigned, or use `let to = ...;` inline.

  3. **`erase__()` line 447**: `let mut has_sibling = false;` is immediately overwritten at the top of the `loop` body at line 451. This variable is set inside the loop and used as a break condition later. Fix: move the declaration inside the loop as `let has_sibling = ...;` since the value is always assigned before use within each iteration.

  4. **`next()` line 559**: `let mut c: u8 = 0;` is overwritten in both `#[cfg(feature = "reduced-trie")]` and `#[cfg(not(feature = "reduced-trie"))]` blocks. Fix: restructure to use a single `let c: u8 = ...;` with cfg-conditional initialization, or use a cfg-match pattern.

## 4. Code Quality & Lint Cleanup

- **`next_until_none`**: Rewrite `while let` (that only loops once) to `if let`, removing `#[allow(clippy::never_loop)]`.
- **`Block`**: Add `Default` impl.
- **Comment typos**: Fix "elemenet" -> "element", "fpr" -> "for", "chilren" -> "children", "finsihed" -> "finished", "doubley" -> "doubly", "ti" -> "to".

## 5. README & Documentation

- Replace Travis CI badge with GitHub Actions badge
- Remove codecov badge if not configured
- Remove Rust 2015 `extern crate` instructions
- Update dependency version in examples to `"0.5"`
- **Fix README example bug**: The current examples call `.iter()` directly on the return value of `common_prefix_search()`, which returns `Option<Vec<...>>`. `Option::iter()` iterates over 0-or-1 references to the inner value, not over the vector elements. The chained `.map(|x| x.0)` would then operate on `&Vec<(i32, usize)>`, not on `(i32, usize)`. Fix the examples to properly unwrap and iterate:
  ```rust
  let result: Vec<i32> = cedar.common_prefix_search("abcdefg")
      .unwrap_or_default()
      .iter()
      .map(|x| x.0)
      .collect();
  ```
- Ensure README examples match `Option` return semantics (document that `None` means no matches)
- Update crate-level doc comment in `src/lib.rs` similarly

## 6. Benchmark Modernization

### Criterion benchmark (`benches/cedarwood_benchmark.rs`)

The benchmark file uses Rust 2015 patterns that must be updated:

- Remove `#[macro_use] extern crate criterion;` and `extern crate cedarwood;`
- Replace with explicit imports: `use criterion::{criterion_group, criterion_main, Criterion};`
- Verify compatibility with criterion 0.5 API (macro usage for `criterion_group!` and `criterion_main!` should still work with the direct imports)

### Macro-benchmark (`benches/macro-benchmark/`)

This is a standalone binary crate with its own `Cargo.toml` (edition 2018, path dependency on cedarwood). Check and update:

- **Edition**: `2018` -> `2021` in `benches/macro-benchmark/Cargo.toml`
- **Path dependency**: Verify that the path dependency `cedarwood = { path = "../../" }` continues to resolve correctly
- **Code**: The `src/main.rs` already uses `use cedarwood::Cedar;` (not `extern crate`), so no import changes needed
- Note: this sub-crate is excluded from the main crate's publish via the existing `exclude = ["/benches/**"]` in `Cargo.toml`, so no publish concerns

## 7. Decisions

| Decision | Choice | Alternatives Considered |
|----------|--------|------------------------|
| Edition | 2021 | 2024 (too new, stricter unsafe linting adds churn) |
| Deps | Moderate update | Conservative (miss criterion 0.5), Aggressive (rand 0.9 breaking changes) |
| Unsafe | Rewrite to safe | Keep with comments, Rewrite + debug_assert |
| Option semantics | None for empty (keep Option wrapper) | Change to Vec (simpler but loses semantic distinction and breaks callers using `.unwrap()`); Keep as-is returning `Some(vec![])` (misleading) |
| CEDAR_VALUE_LIMIT | cfg-gate the constant | Keep `#[allow(dead_code)]` (justified but not as clean), Remove annotation (causes warning in default build) |
| Integer casts | Separate commit for `i32::from()` conversion | Leave as-is, debug_assert guards, TryFrom (perf overhead) |
| Metadata | Full cleanup including .travis.yml deletion | Code only, Minimal |

## 8. Test Oracle

All existing tests must continue to pass after every change. Run `cargo test` before considering any step complete. Run `cargo test --all-features` to verify `reduced-trie` feature still works (especially the cfg-gated `CEDAR_VALUE_LIMIT`). Run `cargo clippy` to verify lint cleanup is effective. Run `cargo test --benches` to verify the benchmark file compiles.

## 9. Optional: CI Workflow Modernization

**Status**: Optional, include if time permits.

The `.github/workflows/CI.yml` uses deprecated/archived actions:

| Current | Replacement |
|---------|-------------|
| `actions/checkout@v2` | `actions/checkout@v4` |
| `actions-rs/toolchain@v1` | `dtolnay/rust-toolchain@stable` |
| `actions-rs/cargo@v1` | Direct `run: cargo ...` commands |
| `actions-rs/tarpaulin@v0.1` | `cargo install cargo-tarpaulin && cargo tarpaulin ...` or a maintained action |
| `actions/upload-artifact@v1` | `actions/upload-artifact@v4` |
| `codecov/codecov-action@v1` | `codecov/codecov-action@v4` |

The `actions-rs/*` actions are unmaintained and archived. Since the design already covers README badge updates and metadata cleanup, modernizing CI is a natural companion change but is not strictly required for the crate itself.

## Implementation Order

1. Cargo.toml metadata, edition bump, delete `.travis.yml`
2. Benchmark file modernization (must happen with or immediately after edition/dep bump)
3. Unsafe code elimination (`pop_sibling` rewrite)
4. Dead code annotation cleanup (`build()`, `pop_sibling`, `CEDAR_VALUE_LIMIT` cfg-gating)
5. Return type semantics (`None` for empty)
6. Lint and code quality cleanup (unused_assignments, never_loop, Default impls, comment typos, constants)
7. Integer cast cleanup (`i32::from()` conversion) -- **separate commit**
8. README and documentation updates
9. (Optional) CI workflow modernization
10. Final verification: `cargo test --all-features`, `cargo clippy --all-features`, `cargo test --benches`
