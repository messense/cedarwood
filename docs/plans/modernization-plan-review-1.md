VERDICT: NEEDS_REVISION

## Summary Assessment

The plan is thorough, well-structured, and covers all requirements from the design doc. However, there are several code snippet errors that would cause compile failures if followed literally, and one step (Step 10) bundles four independent refactoring sites that are not bite-sized.

## Critical Issues (must fix)

### 1. Step 10, `follow()`: Code snippet contradicts its own note -- will not compile

The code snippet declares `let to;` (immutable), but in the else branch, `to` is assigned at line 359 and then potentially reassigned at line 362 (`to = self.resolve(...)`). The plan's own note acknowledges this: "Note: `to` must remain `mut` in the else branch since `resolve` may reassign it. Use `let mut to;` with no initial value." The code snippet must be corrected to use `let mut to;` to match the note.

**Fix**: Change the code snippet from `let to;` to `let mut to;`.

### 2. Step 10, `next()`: Code snippet contradicts its own note -- will not compile

The code snippet declares `let c: u8 = ...;` (immutable) for both `#[cfg]` variants, but `c` is reassigned at line 575 inside the `while` loop: `c = self.n_infos[from as usize].sibling;`. The plan's own note says "c is later reassigned in the while loop body, so it still needs to be `let mut c`." The code snippet must use `let mut c: u8` for both `#[cfg]` variants.

**Fix**: Change both `#[cfg]` code snippets from `let c: u8 = ...` to `let mut c: u8 = ...`.

### 3. Step 10 is not bite-sized (2-5 minutes)

Step 10 bundles four independent refactoring sites (`follow()`, `find()`, `erase__()`, `next()`), each requiring careful analysis of variable lifetimes and cfg-conditional compilation. Given the contradictions already present in the code snippets (Issues 1-2 above), this step is error-prone and should be split into four sub-steps with individual verification after each change.

**Fix**: Split Step 10 into 10a-10d, each addressing one function site with its own `cargo test` and `cargo test --all-features` verification. This also makes each change independently revertible.

### 4. Step 12 is not bite-sized (2-5 minutes)

Step 12 requires converting approximately 52 `as i32` cast sites, identifying which are widening (`u8 as i32`) vs narrowing/same-width (`usize as i32`). This is a large mechanical change that could take 15-20 minutes. The plan correctly identifies it as "large mechanical" but still presents it as a single step. The lack of a specific list of sites to change makes it easy to miss a conversion or convert a non-widening cast incorrectly.

**Fix**: Either (a) run `cargo clippy` first to generate the exact list of sites, include that list in the plan, and then execute the conversions; or (b) acknowledge this step takes longer than 5 minutes and note that `cargo clippy` output should be used as the guide rather than manual identification.

### 5. Step 13: Inconsistency with design doc on `.unwrap()` vs `.unwrap_or_default()`

The plan's Step 13 says to fix README examples by adding `.unwrap()` after `common_prefix_search()`. However, the design doc (Section 5) specifies `.unwrap_or_default()` as the fix. For user-facing example code, `.unwrap_or_default()` is safer and more idiomatic since it handles the `None` case gracefully. Using `.unwrap()` in examples teaches users to ignore the `None` case.

**Fix**: Use `.unwrap_or_default()` in README examples (as the design doc specifies), or alternatively use `.unwrap()` only in examples where the result is guaranteed non-empty and add a comment explaining why.

## Suggestions (nice to have)

### A. Step 5 dependency rationale is misleading

The dependency table says Step 6 (Group 4) "depends on Step 5's removal of pop_sibling annotation." But Step 6 handles `CEDAR_VALUE_LIMIT` (line 146) and `build()` (line 285), which are completely independent of `pop_sibling` (line 819). The only real dependency for Step 6 is on Step 1 (edition must be 2021 for `i32::MAX` to be the idiomatic style, though `i32::MAX` has worked since Rust 1.43). Consider correcting the dependency rationale or removing the false dependency.

### B. Group 5 parallelization caveat

Steps 7, 8, and 9 are listed as parallelizable but all modify `src/lib.rs`. This is fine for a single developer executing sequentially with individual commits, but the word "parallelize" could be misleading if interpreted as literal parallel branches. Consider adding a note: "parallelizable in terms of logical independence, but execute sequentially since they modify the same file."

### C. Consider verifying `--all-features` after Step 5

Step 5's pop_sibling rewrite should be verified with `cargo test --all-features` since the `erase__` function (which calls `pop_sibling`) has `#[cfg(feature = "reduced-trie")]` blocks. The current Step 5c only runs `cargo test` (default features).

### D. Step 10, `find()` fix: consider keeping `to` outside loop for reduced-trie

The proposed fix for `find()` moves `to` inside the loop as `let to = ...`. However, with the `reduced-trie` feature, there is a `break` at line 380 that exits the loop, after which `to` is never used again (the function proceeds to the reduced-trie section). The current code has `to` survive the loop but it is never read after the loop. So moving it inside the loop is safe. Just confirming this is correct -- no action needed, but the implementer should verify with `--all-features`.

### E. Step 15 (CI) lacks specific replacement code

Step 15 lists which actions to replace but does not provide the actual YAML content. This makes it harder to execute as a bite-sized task. Consider including the full replacement CI YAML.

## Verified Claims (things you confirmed are correct)

1. **File paths**: All referenced file paths exist and are correct (`.travis.yml` exists at repo root, `src/lib.rs`, `Cargo.toml`, `benches/cedarwood_benchmark.rs`, `benches/macro-benchmark/Cargo.toml`, `benches/macro-benchmark/src/main.rs`, `README.md`, `.github/workflows/CI.yml`).

2. **Line numbers**: All line number references in the plan match the actual source code:
   - `CEDAR_VALUE_LIMIT` at line 147
   - `#[allow(dead_code)]` at lines 146, 285, 819
   - `#[allow(unused_assignments)]` at lines 346, 371, 447, 559
   - `#[allow(clippy::never_loop)]` at line 199
   - `#[allow(clippy::cast_lossless)]` at line 240
   - `#[allow(clippy::new_without_default)]` at line 243
   - All six typo locations verified at correct lines (100, 404, 650, 834, 945, 1011)

3. **Typo content**: All six typos are real and the corrections are accurate ("elemenet" -> "element", "fpr" -> "for", "doubley" -> "doubly", "ti" -> "to", "finsihed" -> "finished", "chilren" -> "children").

4. **`common_prefix_search` current behavior**: The current implementation `self.common_prefix_iter(key).map(Some).collect()` returns `Some(vec![])` for empty results, confirming the plan's TDD tests (Step 7a) will correctly fail before the fix.

5. **`pop_sibling` safe rewrite logic**: The proposed index-based rewrite using `is_child` flag correctly mirrors the original unsafe pointer-chasing logic. Traced through all cases: first-child match, sibling-chain traversal, and target removal.

6. **`CEDAR_VALUE_LIMIT` usage**: Confirmed the constant is only used inside `#[cfg(feature = "reduced-trie")]` blocks (lines 309, 328, 720, 1060), validating the cfg-gating approach.

7. **Cargo.toml contents**: Current values match what the plan claims to replace -- edition "2018", version "0.4.6", smallvec "1.6.1", criterion "0.4.0", rand "0.8.4", `[badges]` section present, `/.travis.yml` in exclude list.

8. **Benchmark file**: `benches/cedarwood_benchmark.rs` does use `#[macro_use] extern crate criterion;` and `extern crate cedarwood;` at lines 1-3, and already has `use cedarwood::Cedar;` and `use criterion::Criterion;` at lines 5-6, confirming the plan's replacement approach is correct.

9. **Macro-benchmark**: `benches/macro-benchmark/src/main.rs` already uses `use cedarwood::Cedar;` (not `extern crate`), confirming only the edition bump in its `Cargo.toml` is needed.

10. **Design coverage**: All nine sections of the design document are covered by the plan's 16 steps. No missing functionality.

11. **`next_until_none` rewrite**: The proposed `if let` replacement is semantically equivalent to the `while let` that never loops (it always returns inside the first iteration). The plan correctly removes the `#[allow(clippy::never_loop)]` annotation.

12. **README badge**: The current README has a Travis CI badge at line 5 and codecov badge at line 6, confirming the plan's replacement targets.

13. **lib.rs doc comment**: The current doc comment at line 7 shows `cedarwood = "0.4"` and line 10 has the Rust 2015 reference, matching the plan's targets for Step 14.
