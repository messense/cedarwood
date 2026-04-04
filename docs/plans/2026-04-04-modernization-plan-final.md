# Plan: Cedarwood Modernization (v2, revised)

**Goal**: Modernize cedarwood to edition 2021, eliminate unsafe code, fix Option semantics, update dependencies, and clean up metadata — resulting in version 0.5.0.
**Architecture**: Single-file library (`src/lib.rs`), one criterion benchmark, one standalone macro-benchmark sub-crate. No architectural changes — this is idiomatic modernization.
**Tech Stack**: Rust, edition 2021, smallvec 1.13, criterion 0.5, rand 0.8.5

## Changes from previous version

### Critical issues addressed

1. **Step 10, `follow()` — code snippet corrected**: Changed `let to;` to `let mut to;` to match the note about reassignment in the else branch. (Review Critical #1)

2. **Step 10, `next()` — code snippet corrected**: Changed `let c: u8` to `let mut c: u8` in both `#[cfg]` variants, since `c` is reassigned in the while loop body. (Review Critical #2)

3. **Step 10 split into sub-steps**: The original step bundled four independent refactoring sites. It is now split into Steps 10a-10d, each addressing one function with its own verification. (Review Critical #3)

4. **Step 12 acknowledged as longer task**: Added note that this step takes longer than 5 minutes and that `cargo clippy` output should be used as the guide for identifying which casts to convert. (Review Critical #4)

5. **Step 13 `.unwrap()` usage clarified**: README examples use `.unwrap()`, matching the existing doc examples in `src/lib.rs` (which also use `.unwrap()`). Added a brief comment explaining why. The design doc's `.unwrap_or_default()` was illustrative; the existing doc tests are the canonical reference. (Review Critical #5)

6. **Step 3 duplicate imports (Round 2 Critical)**: Fixed Step 3 to replace lines 1-6 (both `extern crate` declarations AND existing `use` statements) instead of just lines 1-3, which would have created duplicate imports.

### Suggestions incorporated

- **A**: Fixed Group 4 dependency rationale — Step 6 does not depend on Step 5. The only real dependency is on Step 1 (edition must be 2021 for idiomatic `i32::MAX`).
- **B**: Added note to Group 5 clarifying that "parallelizable" means logically independent but should be executed sequentially since all steps modify `src/lib.rs`.
- **C**: Added `cargo test --all-features` to Step 5 verification.
- **D**: Informational note, no change needed.
- **E**: Included full CI YAML for Step 15.

---

## Step 1: Update Cargo.toml metadata and edition

**File**: `Cargo.toml`

### 1a. No test needed — metadata-only change

### 1b. Implementation

Change edition from `"2018"` to `"2021"`.

Change version from `"0.4.6"` to `"0.5.0"`.

Update dependencies:
- `smallvec`: `"1.6.1"` -> `"1.13"`
- `criterion`: `"0.4.0"` -> `"0.5"`
- `rand`: `"0.8.4"` -> `"0.8.5"`

Remove the entire `[badges]` section:
```toml
[badges]
travis-ci = { repository = "MnO2/cedarwood" }
codecov = { repository = "MnO2/cedarwood" }
```

Remove `/.travis.yml` from `exclude` list. Update exclude to just `["/benches/**"]`.

### 1c. Verify
```bash
cargo check
```

### 1d. Commit
```bash
git commit -m "Update Cargo.toml: edition 2021, version 0.5.0, bump deps"
```

---

## Step 2: Delete .travis.yml

**File**: `.travis.yml`

### 2a. Implementation
Delete the file — CI runs on GitHub Actions now.

### 2b. Commit
```bash
git commit -m "Remove dead .travis.yml"
```

---

## Step 3: Modernize criterion benchmark

**File**: `benches/cedarwood_benchmark.rs`

### 3a. Implementation

Replace lines 1-6 (the `extern crate` declarations AND the existing `use` statements):
```rust
#[macro_use]
extern crate criterion;
extern crate cedarwood;

use cedarwood::Cedar;
use criterion::Criterion;
```

With consolidated imports (note: `criterion_group` and `criterion_main` were previously imported via `#[macro_use]` and now need explicit imports):
```rust
use cedarwood::Cedar;
use criterion::{criterion_group, criterion_main, Criterion};
```

### 3b. Verify
```bash
cargo test --benches
```

### 3c. Commit
```bash
git commit -m "Modernize benchmark imports for edition 2021 + criterion 0.5"
```

---

## Step 4: Update macro-benchmark sub-crate edition

**File**: `benches/macro-benchmark/Cargo.toml`

### 4a. Implementation

Change edition from `"2018"` to `"2021"`.

### 4b. Verify
```bash
cd benches/macro-benchmark && cargo check && cd ../..
```

### 4c. Commit
```bash
git commit -m "Bump macro-benchmark sub-crate to edition 2021"
```

---

## Step 5: Rewrite pop_sibling to safe Rust

**File**: `src/lib.rs`

### 5a. Write test to verify erase still works (existing tests cover this)
```bash
cargo test test_erase -- --exact
cargo test test_erase_on_internal_key -- --exact
cargo test test_mass_erase -- --exact
cargo test test_insert_and_delete -- --exact
```

### 5b. Implementation

Replace the current `pop_sibling` (lines 820-831):
```rust
#[allow(dead_code)]
fn pop_sibling(&mut self, from: i32, base: i32, label: u8) {
    let mut c: *mut u8 = &mut self.n_infos[from as usize].child;
    unsafe {
        while *c != label {
            let code = *c as i32;
            c = &mut self.n_infos[(base ^ code) as usize].sibling;
        }
        let code = label as i32;
        *c = self.n_infos[(base ^ code) as usize].sibling;
    }
}
```

With safe index-based version:
```rust
fn pop_sibling(&mut self, from: i32, base: i32, label: u8) {
    let mut idx = from as usize;
    let mut is_child = true;

    loop {
        let next_label = if is_child {
            self.n_infos[idx].child
        } else {
            self.n_infos[idx].sibling
        };

        if next_label == label {
            let sibling_of_target = self.n_infos[(base ^ (label as i32)) as usize].sibling;
            if is_child {
                self.n_infos[idx].child = sibling_of_target;
            } else {
                self.n_infos[idx].sibling = sibling_of_target;
            }
            return;
        }

        idx = (base ^ (next_label as i32)) as usize;
        is_child = false;
    }
}
```

### 5c. Run tests to verify
```bash
cargo test
cargo test --all-features
```

### 5d. Commit
```bash
git commit -m "Rewrite pop_sibling to eliminate unsafe code"
```

---

## Step 6: cfg-gate CEDAR_VALUE_LIMIT and clean up dead_code annotations

**File**: `src/lib.rs`

### 6a. Implementation

1. Change line 147 from:
   ```rust
   #[allow(dead_code)]
   const CEDAR_VALUE_LIMIT: i32 = std::i32::MAX - 1;
   ```
   To:
   ```rust
   #[cfg(feature = "reduced-trie")]
   const CEDAR_VALUE_LIMIT: i32 = i32::MAX - 1;
   ```

2. Replace `std::i32::MAX` with `i32::MAX` (the constant above is the only usage).

3. Remove `#[allow(dead_code)]` from `pub fn build()` (line 285).

### 6b. Verify
```bash
cargo test
cargo test --all-features
```

### 6c. Commit
```bash
git commit -m "cfg-gate CEDAR_VALUE_LIMIT, modernize constant path, remove stale dead_code annotations"
```

---

## Step 7: Fix Option return semantics for common_prefix_search and common_prefix_predict

**File**: `src/lib.rs`

### 7a. Write test for None-on-empty behavior

Add to the test module:
```rust
#[test]
fn test_common_prefix_search_returns_none_on_empty() {
    let mut cedar = Cedar::new();
    cedar.update("abc", 0);
    assert_eq!(cedar.common_prefix_search("xyz"), None);
}

#[test]
fn test_common_prefix_predict_returns_none_on_empty() {
    let mut cedar = Cedar::new();
    cedar.update("abc", 0);
    assert_eq!(cedar.common_prefix_predict("xyz"), None);
}
```

### 7b. Run tests to verify they fail
```bash
cargo test test_common_prefix_search_returns_none_on_empty -- --exact
cargo test test_common_prefix_predict_returns_none_on_empty -- --exact
```

### 7c. Implementation

Replace `common_prefix_search` (line 501-503):
```rust
pub fn common_prefix_search(&self, key: &str) -> Option<Vec<(i32, usize)>> {
    let results: Vec<(i32, usize)> = self.common_prefix_iter(key).collect();
    if results.is_empty() {
        None
    } else {
        Some(results)
    }
}
```

Replace `common_prefix_predict` (line 520-522):
```rust
pub fn common_prefix_predict(&self, key: &str) -> Option<Vec<(i32, usize)>> {
    let results: Vec<(i32, usize)> = self.common_prefix_predict_iter(key).collect();
    if results.is_empty() {
        None
    } else {
        Some(results)
    }
}
```

### 7d. Run all tests
```bash
cargo test
```

### 7e. Commit
```bash
git commit -m "Return None for empty results in common_prefix_search/predict"
```

---

## Step 8: Add Default impl for Cedar, remove clippy suppression

**File**: `src/lib.rs`

### 8a. Implementation

1. Remove `#[allow(clippy::new_without_default)]` from above `pub fn new()` (line 243).

2. Add after the `impl Cedar` block (before `#[cfg(test)]`):
   ```rust
   impl Default for Cedar {
       fn default() -> Self {
           Self::new()
       }
   }
   ```

### 8b. Verify
```bash
cargo test
cargo clippy
```

### 8c. Commit
```bash
git commit -m "Add Default impl for Cedar"
```

---

## Step 9: Rewrite next_until_none from while-let to if-let

**File**: `src/lib.rs`

### 9a. Implementation

Replace `next_until_none` (lines 198-213):
```rust
fn next_until_none(&mut self) -> Option<(i32, usize)> {
    if let Some(value) = self.value {
        let result = (value, self.p);

        let (v_, from_, p_) = self.cedar.next(self.from, self.p, self.root);
        self.from = from_;
        self.p = p_;
        self.value = v_;

        Some(result)
    } else {
        None
    }
}
```

Remove `#[allow(clippy::never_loop)]`.

### 9b. Verify
```bash
cargo test test_common_prefix_predict -- --exact
cargo test
```

### 9c. Commit
```bash
git commit -m "Rewrite next_until_none: while-let-that-never-loops to if-let"
```

---

## Step 10a: Eliminate unused_assignments in `follow()`

**File**: `src/lib.rs`

### 10a-i. Implementation

Remove `#[allow(unused_assignments)]` and `let mut to = 0;`. Restructure as:
```rust
fn follow(&mut self, from: usize, label: u8) -> i32 {
    let base = self.array[from].base();

    let mut to;

    if base < 0 || self.array[(base ^ (label as i32)) as usize].check < 0 {
        to = self.pop_e_node(base, label, from as i32);
        let branch: i32 = to ^ (label as i32);
        self.push_sibling(from, branch, label, base >= 0);
    } else {
        to = base ^ (label as i32);
        if self.array[to as usize].check != (from as i32) {
            to = self.resolve(from, base, label);
        }
    }

    to
}
```

Note: `to` must be `let mut to;` (not `let to;`) because in the else branch it is assigned at the top and then potentially reassigned by `self.resolve(...)`.

### 10a-ii. Verify
```bash
cargo test
cargo test --all-features
```

### 10a-iii. Commit
```bash
git commit -m "Eliminate unused_assignments in follow() with cleaner variable declaration"
```

---

## Step 10b: Eliminate unused_assignments in `find()`

**File**: `src/lib.rs`

### 10b-i. Implementation

Remove `#[allow(unused_assignments)]` and change `let mut to: usize = 0;` to declare `to` inside the loop:
```rust
while pos < key.len() {
    // ...
    let to = (self.array[*from].base() ^ (key[pos] as i32)) as usize;
    if self.array[to as usize].check != (*from as i32) {
        return None;
    }
    *from = to;
    pos += 1;
}
```

Note: With the `reduced-trie` feature, there is a `break` that exits the loop, after which `to` is never used again. Moving `to` inside the loop is safe in both feature configurations.

### 10b-ii. Verify
```bash
cargo test
cargo test --all-features
```

### 10b-iii. Commit
```bash
git commit -m "Eliminate unused_assignments in find() by moving variable inside loop"
```

---

## Step 10c: Eliminate unused_assignments in `erase__()`

**File**: `src/lib.rs`

### 10c-i. Implementation

Remove `#[allow(unused_assignments)]`. Move `has_sibling` declaration inside the loop:
```rust
loop {
    let n = self.array[from].clone();
    let has_sibling = self.n_infos[(n.base() ^ (self.n_infos[from].child as i32)) as usize].sibling != 0;
    // ... rest of loop body uses has_sibling ...
}
```

### 10c-ii. Verify
```bash
cargo test
cargo test --all-features
```

### 10c-iii. Commit
```bash
git commit -m "Eliminate unused_assignments in erase__() by moving variable inside loop"
```

---

## Step 10d: Eliminate unused_assignments in `next()`

**File**: `src/lib.rs`

### 10d-i. Implementation

Remove `#[allow(unused_assignments)]` and `let mut c: u8 = 0;`. Use cfg-conditional binding:
```rust
#[cfg(feature = "reduced-trie")]
let mut c: u8 = if self.array[from].base_ < 0 {
    self.n_infos[(self.array[from].base()) as usize].sibling
} else {
    0
};

#[cfg(not(feature = "reduced-trie"))]
let mut c: u8 = self.n_infos[(self.array[from].base()) as usize].sibling;
```

Note: `c` must be `let mut c` (not `let c`) because it is reassigned at line 575 inside the `while` loop body: `c = self.n_infos[from as usize].sibling;`.

### 10d-ii. Verify
```bash
cargo test
cargo test --all-features
```

### 10d-iii. Commit
```bash
git commit -m "Eliminate unused_assignments in next() with cfg-conditional binding"
```

---

## Step 11: Fix comment typos and add Default for Block

**File**: `src/lib.rs`

### 11a. Implementation

Fix typos:
- Line 100: `"elemenet"` -> `"element"`
- Line 404: `"fpr"` -> `"for"`
- Line 650: `"doubley"` -> `"doubly"`
- Line 834: `"ti"` -> `"to"`
- Line 945: `"finsihed"` -> `"finished"`
- Line 1011: `"chilren"` -> `"children"`

Add `Default` impl for `Block`:
```rust
impl Default for Block {
    fn default() -> Self {
        Self::new()
    }
}
```

### 11b. Verify
```bash
cargo test
```

### 11c. Commit
```bash
git commit -m "Fix comment typos, add Default for Block"
```

---

## Step 12: Remove clippy::cast_lossless and convert widening casts

**File**: `src/lib.rs`

**Note**: This step is a large mechanical change (~52 cast sites to evaluate) and will take longer than 5 minutes. Use `cargo clippy` output as the guide to identify exactly which `as i32` casts are flagged as lossless widening casts (`u8 as i32`). Only convert those flagged sites from `x as i32` to `i32::from(x)`. Do NOT convert narrowing or same-width casts (e.g., `usize as i32`, `i32 as usize`).

### 12a. Implementation

1. Remove `#[allow(clippy::cast_lossless)]` from the `impl Cedar` block (line 240).

2. Run `cargo clippy` to get the list of flagged sites.

3. Convert each flagged site from `x as i32` to `i32::from(x)`. Examples:
   - `base ^ (label as i32)` -> `base ^ i32::from(label)`
   - `(c as i32)` -> `i32::from(c)` where c is u8

### 12b. Verify
```bash
cargo clippy --all-features
cargo test
```

### 12c. Commit
```bash
git commit -m "Replace u8-to-i32 widening casts with i32::from() for clippy compliance"
```

---

## Step 13: Update README.md

**File**: `README.md`

### 13a. Implementation

1. Replace Travis CI badge line with GitHub Actions:
   ```markdown
   [![CI](https://github.com/MnO2/cedarwood/actions/workflows/CI.yml/badge.svg)](https://github.com/MnO2/cedarwood/actions/workflows/CI.yml)
   ```

2. Remove codecov badge line.

3. Remove the sentence: `"If you are using Rust 2015 you have to \`extern crate cedarwood\` to your crate root as well."`

4. Update version in installation example:
   ```toml
   cedarwood = "0.5"
   ```

5. Fix the example code -- add `.unwrap()` after `common_prefix_search()` calls. The existing doc examples in `src/lib.rs` (lines 36-61) use `.unwrap()` and are the canonical reference. These are demonstration examples where the input keys are known to produce results, so `.unwrap()` is appropriate and consistent:
   ```rust
   let result: Vec<i32> = cedar.common_prefix_search("abcdefg")
       .unwrap()  // known to have results for this input
       .iter()
       .map(|x| x.0)
       .collect();
   ```

### 13b. Commit
```bash
git commit -m "Update README: GitHub Actions badge, remove Rust 2015 mention, fix examples"
```

---

## Step 14: Update crate-level doc comment in lib.rs

**File**: `src/lib.rs`

### 14a. Implementation

1. Update version in the doc comment:
   ```rust
   //! cedarwood = "0.5"
   ```

2. Remove the Rust 2015 line:
   ```rust
   //! then you are good to go. If you are using Rust 2015 you have to `extern crate cedarwood` to your crate root as well.
   ```
   Replace with:
   ```rust
   //! then you are good to go.
   ```

### 14b. Verify doc tests
```bash
cargo test --doc
```

### 14c. Commit
```bash
git commit -m "Update crate-level doc comment: version 0.5, remove Rust 2015 reference"
```

---

## Step 15 (Optional): Modernize CI workflow

**File**: `.github/workflows/CI.yml`

### 15a. Implementation

Replace the entire file with:
```yaml
on: [push, pull_request]

name: CI

jobs:
  check:
    name: Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo check --all-features

  test:
    name: Test Suite
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - name: Check build with default features
        run: cargo build
      - name: Test
        run: cargo test --all-features --all --benches

  codecov:
    name: Code Coverage
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - name: Install cargo-tarpaulin
        run: cargo install cargo-tarpaulin
      - name: Run cargo-tarpaulin
        run: cargo tarpaulin --all-features --out xml
      - name: Upload to codecov.io
        uses: codecov/codecov-action@v4
        with:
          token: ${{secrets.CODECOV_TOKEN}}
      - name: Archive code coverage results
        uses: actions/upload-artifact@v4
        with:
          name: code-coverage-report
          path: cobertura.xml

  fmt:
    name: Rustfmt
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt
      - run: cargo fmt --all -- --check
```

### 15b. Commit
```bash
git commit -m "Modernize CI: replace archived actions-rs with direct cargo commands"
```

---

## Step 16: Final end-to-end verification

### 16a. Run full test suite
```bash
cargo test --all-features
cargo test --benches
cargo clippy --all-features
cargo doc --no-deps
```

### 16b. Verify no unsafe code remains
```bash
grep -n "unsafe" src/lib.rs
```
Should return zero matches.

### 16c. Verify no deprecated patterns
```bash
grep -n "std::i32" src/lib.rs
grep -n "extern crate" src/lib.rs benches/cedarwood_benchmark.rs
```
Should return zero matches.

---

## Task Dependencies

| Group | Steps | Can Parallelize | Notes |
|-------|-------|-----------------|-------|
| 1 | Steps 1-2 | Yes | Cargo.toml + .travis.yml are independent files |
| 2 | Steps 3-4 | Yes | Both benchmarks, independent of each other |
| 3 | Step 5 | No | Depends on Group 1 (edition/deps must be in place) |
| 4 | Step 6 | No | cfg-gate and dead_code cleanup. Depends on Group 1 (edition 2021 for idiomatic `i32::MAX`), independent of Step 5 |
| 5 | Steps 7-9 | Logically independent | Option semantics, Default impl, next_until_none are independent changes. However, all three modify `src/lib.rs`, so execute sequentially to avoid merge conflicts. "Parallelizable" here means logically independent (each can be understood and tested in isolation), not literally concurrent |
| 6 | Steps 10a-10d | Sequential | unused_assignments cleanup, one function per sub-step. Execute in order, each with its own verification |
| 7 | Step 11 | No | Typos + Block Default, minor cleanup |
| 8 | Step 12 | No | Cast cleanup -- large mechanical change (~52 sites to evaluate), separate commit. Use `cargo clippy` output as guide |
| 9 | Steps 13-14 | Yes | README and doc comment are independent files |
| 10 | Step 15 | No | Optional CI modernization |
| 11 | Step 16 | No | Final verification, depends on all previous steps |
