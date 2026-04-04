# Cedarwood Modernization Design

**Date**: 2026-04-04
**Version**: 0.4.6 → 0.5.0 (semver major bump due to semantic change)

## Goal

Modernize the cedarwood crate to use current Rust idioms, fix potential bugs, eliminate unsafe code, and refresh project metadata. The library's behavior and public API signatures remain unchanged, but the semantics of `Option` return values are corrected.

## 1. Cargo.toml & Metadata

- **Edition**: `2018` → `2021`
- **Version**: `0.4.6` → `0.5.0`
- **Dependencies**:
  - `smallvec`: `"1.6.1"` → `"1.13"` (latest 1.x minor)
  - `criterion` (dev): `"0.4.0"` → `"0.5"` (dev-only, minor API changes)
  - `rand` (dev): `"0.8.4"` → `"0.8.5"` (latest patch)
- **Remove**: `[badges]` section (dead Travis CI / codecov entries)
- **Remove**: `/.travis.yml` from `exclude` list

## 2. Unsafe Code Elimination

Rewrite `pop_sibling` from raw pointer (`*mut u8`) traversal to safe index-based traversal using a boolean `is_child` flag to track whether we're reading/writing the `child` or `sibling` field. Same logic, zero `unsafe`.

Remove `#[allow(dead_code)]` from `pop_sibling` since it is called from `erase__`.

## 3. Return Type Semantics & API Fixes

- **`common_prefix_search` / `common_prefix_predict`**: Keep `Option<Vec<(i32, usize)>>` signature but return `None` for empty results instead of `Some(vec![])`. This is a semantic breaking change (hence semver major bump).
- **`Default` for `Cedar`**: Add a proper `Default` impl delegating to `Cedar::new()`, remove `#[allow(clippy::new_without_default)]`.
- **Constants**: Replace `std::i32::MAX` with `i32::MAX`.
- **Dead code annotations**: Remove `#[allow(dead_code)]` from `CEDAR_VALUE_LIMIT` and `pop_sibling` — audit actual usage.
- **Cast lint**: Remove `#[allow(clippy::cast_lossless)]` from impl block, fix individual sites if clippy flags them.
- **Unused assignments**: Restructure variable declarations to eliminate `#[allow(unused_assignments)]` where possible.

## 4. Code Quality & Lint Cleanup

- **`next_until_none`**: Rewrite `while let` (that only loops once) to `if let`, removing `#[allow(clippy::never_loop)]`.
- **`Block`**: Add `Default` impl.
- **Comment typos**: Fix "elemenet" → "element", "fpr" → "for", "chilren" → "children", "finsihed" → "finished", "doubley" → "doubly", "ti" → "to".

## 5. README & Documentation

- Replace Travis CI badge with GitHub Actions badge
- Remove codecov badge if not configured
- Remove Rust 2015 `extern crate` instructions
- Update dependency version in examples to `"0.5"`
- Ensure README examples match `Option` return semantics (`.unwrap()` calls)
- Update crate-level doc comment in `src/lib.rs` similarly

## Decisions

| Decision | Choice | Alternatives Considered |
|----------|--------|------------------------|
| Edition | 2021 | 2024 (too new, stricter unsafe linting adds churn) |
| Deps | Moderate update | Conservative (miss criterion 0.5), Aggressive (rand 0.9 breaking changes) |
| Unsafe | Rewrite to safe | Keep with comments, Rewrite + debug_assert |
| Option semantics | None for empty | Change to Vec (breaks signature), Keep as-is (misleading) |
| Integer casts | Leave as-is | debug_assert guards, TryFrom (perf overhead) |
| Metadata | Full cleanup | Code only, Minimal |

## Test Oracle

All existing tests must continue to pass after every change. Run `cargo test` before considering any step complete. Run `cargo clippy` to verify lint cleanup is effective.
