# API Reference

This document covers every public type and method in cedarwood. For the auto-generated rustdoc, see [docs.rs/cedarwood](https://docs.rs/cedarwood/).

## `Cedar`

The main trie data structure. Holds the double-array, node metadata, and block management state.

```rust
pub struct Cedar { /* private fields */ }
```

`Cedar` implements `Clone`, `Default`, and `Debug`.

---

### `Cedar::new()`

```rust
pub fn new() -> Self
```

Creates an empty trie. The internal array is pre-allocated with one block (256 slots). The root node is at index 0.

**Example:**

```rust
use cedarwood::Cedar;
let cedar = Cedar::new();
```

---

### `Cedar::build()`

```rust
pub fn build(&mut self, key_values: &[(&str, i32)])
```

Bulk-inserts a slice of (key, value) pairs. Equivalent to calling `update` for each pair. Keys do not need to be sorted, but providing them in sorted order may result in slightly better memory layout.

**Panics** if any key is empty (zero-length).

**Example:**

```rust
use cedarwood::Cedar;

let entries = vec![("alpha", 1), ("beta", 2), ("gamma", 3)];
let mut cedar = Cedar::new();
cedar.build(&entries);
```

---

### `Cedar::update()`

```rust
pub fn update(&mut self, key: &str, value: i32)
```

Inserts a single key with the given value, or overwrites the value if the key already exists.

**Panics** if `key` is empty.

**Example:**

```rust
use cedarwood::Cedar;

let mut cedar = Cedar::new();
cedar.update("hello", 42);

// Overwrite the value
cedar.update("hello", 99);
assert_eq!(cedar.exact_match_search("hello").map(|x| x.0), Some(99));
```

---

### `Cedar::erase()`

```rust
pub fn erase(&mut self, key: &str)
```

Deletes a key from the trie. If the key does not exist, this is a no-op. After deletion, `exact_match_search` for that key returns `None`.

**Example:**

```rust
use cedarwood::Cedar;

let mut cedar = Cedar::new();
cedar.update("hello", 42);
cedar.erase("hello");
assert_eq!(cedar.exact_match_search("hello"), None);
```

---

### `Cedar::exact_match_search()`

```rust
pub fn exact_match_search(&self, key: &str) -> Option<(i32, usize, usize)>
```

Looks up `key` in the trie. Returns `Some((value, key_length, node_position))` if found, `None` otherwise.

| Return field | Description |
|-------------|-------------|
| `value` (i32) | The stored value |
| `key_length` (usize) | Length of the matched key in bytes |
| `node_position` (usize) | Internal node index (useful for advanced traversal) |

**Example:**

```rust
use cedarwood::Cedar;

let mut cedar = Cedar::new();
cedar.update("abc", 10);

match cedar.exact_match_search("abc") {
    Some((value, len, _pos)) => {
        assert_eq!(value, 10);
        assert_eq!(len, 3);
    }
    None => panic!("key not found"),
}

assert_eq!(cedar.exact_match_search("xyz"), None);
```

---

### `Cedar::common_prefix_search()`

```rust
pub fn common_prefix_search(&self, key: &str) -> Option<Vec<(i32, usize)>>
```

Finds all keys in the trie that are prefixes of `key`. Returns `None` if no prefix matches are found, or `Some(vec)` where each element is `(value, byte_position)`.

The `byte_position` indicates the byte index of the **last byte** of the matched prefix within `key`.

**Example:**

```rust
use cedarwood::Cedar;

let mut cedar = Cedar::new();
let entries = vec![("a", 0), ("ab", 1), ("abc", 2), ("b", 3)];
cedar.build(&entries);

// Query "abcdef": matches "a" (0), "ab" (1), "abc" (2)
let results = cedar.common_prefix_search("abcdef").unwrap();
let values: Vec<i32> = results.iter().map(|x| x.0).collect();
assert_eq!(values, vec![0, 1, 2]);

// No prefix match
assert_eq!(cedar.common_prefix_search("xyz"), None);
```

---

### `Cedar::common_prefix_iter()`

```rust
pub fn common_prefix_iter<'a>(&'a self, key: &'a str) -> PrefixIter<'a>
```

Returns an iterator over prefix matches. This is the lazy/streaming version of `common_prefix_search` -- it yields matches one at a time without collecting into a `Vec`. Useful when you only need the first few matches or want to short-circuit.

**Example:**

```rust
use cedarwood::Cedar;

let mut cedar = Cedar::new();
cedar.build(&vec![("a", 0), ("ab", 1), ("abc", 2)]);

// Take only the first match
let first = cedar.common_prefix_iter("abcdef").next();
assert_eq!(first, Some((0, 0))); // value=0, position=0
```

---

### `Cedar::common_prefix_predict()`

```rust
pub fn common_prefix_predict(&self, key: &str) -> Option<Vec<(i32, usize)>>
```

Finds all keys in the trie that **start with** `key` as a prefix. This is the reverse of `common_prefix_search`: instead of finding dictionary keys that are prefixes of the query, it finds dictionary keys that the query is a prefix of.

Returns `None` if no keys match, or `Some(vec)` where each element is `(value, depth)`. The `depth` indicates how deep in the subtree the match was found.

**Example:**

```rust
use cedarwood::Cedar;

let mut cedar = Cedar::new();
cedar.build(&vec![("app", 0), ("apple", 1), ("application", 2), ("bat", 3)]);

// Find all keys starting with "app"
let results = cedar.common_prefix_predict("app").unwrap();
let values: Vec<i32> = results.iter().map(|x| x.0).collect();
assert_eq!(values, vec![0, 1, 2]); // "app", "apple", "application"

// No keys start with "xyz"
assert_eq!(cedar.common_prefix_predict("xyz"), None);
```

---

### `Cedar::common_prefix_predict_iter()`

```rust
pub fn common_prefix_predict_iter<'a>(&'a self, key: &'a str) -> PrefixPredictIter<'a>
```

Iterator version of `common_prefix_predict`. Yields matches lazily.

**Example:**

```rust
use cedarwood::Cedar;

let mut cedar = Cedar::new();
cedar.build(&vec![("a", 0), ("ab", 1), ("abc", 2)]);

let count = cedar.common_prefix_predict_iter("a").count();
assert_eq!(count, 3);
```

---

## `PrefixIter<'a>`

```rust
pub struct PrefixIter<'a> { /* private fields */ }
```

Iterator returned by `common_prefix_iter`. Yields `(i32, usize)` tuples of `(value, byte_position)`.

Implements `Iterator` and `Clone`.

**`size_hint`**: returns `(0, Some(key.len()))` -- there can be at most one match per byte position.

---

## `PrefixPredictIter<'a>`

```rust
pub struct PrefixPredictIter<'a> { /* private fields */ }
```

Iterator returned by `common_prefix_predict_iter`. Yields `(i32, usize)` tuples of `(value, depth)`.

Implements `Iterator` and `Clone`.

---

## Feature Flags

| Flag | Default | Description |
|------|---------|-------------|
| `reduced-trie` | off | Stores values directly in leaf nodes instead of using terminal nodes. Reduces node count but adds complexity for keys that are prefixes of other keys. |

## Value Constraints

- Keys must be valid UTF-8 strings (enforced by the `&str` type).
- Keys must be non-empty. Inserting an empty key panics.
- Values are `i32`. The value `-1` is reserved internally as `CEDAR_NO_VALUE` (sentinel for "no value stored"). Storing `-1` as a user value will cause `exact_match_search` to return `None` for that key.
- With `reduced-trie` enabled, `i32::MAX - 1` is also reserved as `CEDAR_VALUE_LIMIT`.
