# cedarwood

Efficiently-updatable double-array trie in Rust, ported from the C++ [cedar](http://www.tkl.iis.u-tokyo.ac.jp/~ynaga/cedar/) library by Naoki Yoshinaga.

[![CI](https://github.com/MnO2/cedarwood/actions/workflows/CI.yml/badge.svg)](https://github.com/MnO2/cedarwood/actions/workflows/CI.yml)
[![Crates.io](https://img.shields.io/crates/v/cedarwood.svg)](https://crates.io/crates/cedarwood)
[![docs.rs](https://docs.rs/cedarwood/badge.svg)](https://docs.rs/cedarwood/)

## Features

- **Fast lookups** -- double-array tries offer O(k) lookup time where k is the length of the key, with excellent cache locality compared to pointer-based tries.
- **Dynamic updates** -- keys can be inserted and deleted after the initial build, unlike many static trie implementations.
- **Common-prefix search** -- find all keys that are prefixes of a given query string, useful for tokenization and morphological analysis.
- **Predictive search** -- find all keys that share a given prefix, useful for autocomplete.
- **Full Unicode support** -- works with any valid UTF-8 string, including CJK characters, supplementary planes (SIP), and combining characters.
- **Reduced-trie mode** -- optional `reduced-trie` feature flag for a more compact representation at the cost of some flexibility.

## Installation

Add it to your `Cargo.toml`:

```toml
[dependencies]
cedarwood = "0.5"
```

To enable the reduced-trie optimization:

```toml
[dependencies]
cedarwood = { version = "0.5", features = ["reduced-trie"] }
```

## Quick Start

```rust
use cedarwood::Cedar;

fn main() {
    let dict = vec![
        "a", "ab", "abc",
        "网", "网球", "网球拍",
        "中", "中华", "中华人民", "中华人民共和国",
    ];
    let key_values: Vec<(&str, i32)> = dict
        .into_iter()
        .enumerate()
        .map(|(k, s)| (s, k as i32))
        .collect();

    let mut cedar = Cedar::new();
    cedar.build(&key_values);

    // Exact match
    let result = cedar.exact_match_search("中华人民");
    assert!(result.is_some());

    // Common prefix search: finds "网", "网球", "网球拍"
    let result: Vec<i32> = cedar
        .common_prefix_search("网球拍卖会")
        .unwrap()
        .iter()
        .map(|x| x.0)
        .collect();
    assert_eq!(vec![3, 4, 5], result);

    // Predictive search: finds all keys starting with "中"
    let result: Vec<i32> = cedar
        .common_prefix_predict("中")
        .unwrap()
        .iter()
        .map(|x| x.0)
        .collect();
    assert_eq!(vec![6, 7, 8, 9], result);
}
```

## API Overview

| Method | Description |
|--------|-------------|
| `Cedar::new()` | Create an empty trie |
| `build(&key_values)` | Bulk-insert key-value pairs |
| `update(key, value)` | Insert or update a single key |
| `erase(key)` | Delete a key from the trie |
| `exact_match_search(key)` | Look up an exact key, returns `Option<(value, length, node_pos)>` |
| `common_prefix_search(key)` | Find all dictionary keys that are prefixes of `key` |
| `common_prefix_iter(key)` | Iterator version of `common_prefix_search` |
| `common_prefix_predict(key)` | Find all dictionary keys that start with `key` |
| `common_prefix_predict_iter(key)` | Iterator version of `common_prefix_predict` |

For detailed API documentation, see [docs.rs](https://docs.rs/cedarwood/) or the [docs/](docs/) folder.

## Use Cases

- **Text segmentation / tokenization** -- common-prefix search is the core operation for dictionary-based Chinese/Japanese word segmentation.
- **Autocomplete / suggest** -- predictive search returns all completions for a typed prefix.
- **Morphological analysis** -- fast dictionary lookup for NLP pipelines.
- **IP routing tables** -- longest-prefix matching on serialized address bytes.
- **Keyword filtering** -- scan text for occurrences of any keyword in a large dictionary.

## Benchmarks

```bash
cargo bench
```

A macro-benchmark with a real dictionary is available in `benches/macro-benchmark/`. A C++ cedar benchmark for comparison is in `benches/cpp/`.

## Documentation

- [Architecture & Design](docs/architecture.md) -- how the double-array trie works
- [API Reference](docs/api-reference.md) -- detailed method documentation
- [Internal Implementation](docs/internals.md) -- block management, conflict resolution, and memory layout

## License

This work is released under the BSD-2-Clause license, following the original license of C++ cedar. A copy of the license is provided in the [LICENSE](LICENSE) file.

## Reference

- [cedar -- C++ implementation of efficiently-updatable double-array trie](http://www.tkl.iis.u-tokyo.ac.jp/~ynaga/cedar/) by Naoki Yoshinaga
- Aoe, J. (1989). [An efficient implementation of trie structures](https://dl.acm.org/citation.cfm?id=146691). *Software: Practice and Experience*.
