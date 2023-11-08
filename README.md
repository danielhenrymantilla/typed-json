# Typed JSON &emsp; [![Latest Version]][crates.io]

[Latest Version]: https://img.shields.io/crates/v/typed-json.svg
[crates.io]: https://crates.io/crates/typed-json

Typed JSON provides a `json!` macro to build an `impl serde::Serialize`
type with very natural JSON syntax.

```rust
use typed_json::json;

// The type of `john` is `impl serde::Serialize`
let john = json!({
    "name": "John Doe",
    "age": 43,
    "phones": [
        "+44 1234567",
        "+44 2345678"
    ]
});

// Convert to a string of JSON and print it out
println!("{}", serde_json::to_string(&john).unwrap());
```

One neat thing about the `json!` macro is that variables and expressions can
be interpolated directly into the JSON value as you are building it. Serde
will check at compile time that the value you are interpolating is able to
be represented as JSON.

```rust
let full_name = "John Doe";
let age_last_year = 42;

// The type of `john` is `impl serde::Serialize`
let john = json!({
    "name": full_name,
    "age": age_last_year + 1,
    "phones": [
        format!("+44 {}", random_phone())
    ]
});
```

# Comparison to `serde_json`

This crate provides a typed version of `serde_json::json!()`. What does that mean? It means it performs 0 allocations and it creates
a custom type for the JSON object you are representing. For one-off JSON documents, this ends up being considerably faster to encode.
This is 100% compatible with `serde_json::json!` syntax as of `serde_json = "1.0.108"`.

## Benchmark

The following benchmarks indicate serializing a complex deeply-nested JSON document to a `String`.
The `typed_json_core` benchmark uses `serde-json-core` to encode to a `heapless::String`.

```
Timer precision: 41 ns
serialize_string    fastest       │ slowest       │ median        │ mean          │ samples │ iters
├─ serde_json       707 ns        │ 36.62 µs      │ 791 ns        │ 1.096 µs      │ 10000   │ 10000
├─ typed_json       154 ns        │ 844.1 ns      │ 163.1 ns      │ 163.5 ns      │ 10000   │ 320000
╰─ typed_json_core  215.2 ns      │ 742.5 ns      │ 229.5 ns      │ 229.7 ns      │ 10000   │ 320000
```

> Note: The benchmarks use `serde_json::to_string` as it's significantly faster than the `ToString`/`Display` implementation,
> both for `serde_json::json` and `typed_json::json`

# No-std support

It is possible to use `typed_json` with only `core`. Disable the default "std"
feature:

```toml
[dependencies]
typed_json = { version = "0.1", default-features = false }
```

To encode the `Serialize` type to JSON:

you will either need [`serde_json`](https://docs.rs/serde_json/latest/serde_json/index.html) with the `alloc` feature

```toml
[dependencies]
serde_json = { version = "1.0", default-features = false, features = ["alloc"] }
```

or [`serde-json-core`](https://docs.rs/serde-json-core/latest/serde_json_core/index.html) with no dependency on `alloc`

```toml
[dependencies]
serde-json-core = "0.5.1"
```

# Compile time benchmarks

There's no such thing as a true zero-cost abstraction.

I measured the compile times using the large service JSON from <https://kubernetesjsonschema.dev/>.

## Many small documents

In this test, I have split the above JSON file into 31 reasonably-sized documents

### Debug

```sh
$ hyperfine \
    --command-name "typed_json" \
    "pushd tests/crates/stress3 && touch src/main.rs && cargo build" \
    --command-name "serde_json" \
    "pushd tests/crates/stress4 && touch src/main.rs && cargo build"

Benchmark 1: typed_json
  Time (mean ± σ):     132.2 ms ±   3.7 ms    [User: 127.9 ms, System: 73.4 ms]
  Range (min … max):   126.3 ms … 140.9 ms    22 runs

Benchmark 2: serde_json
  Time (mean ± σ):     145.7 ms ±   3.6 ms    [User: 131.9 ms, System: 97.5 ms]
  Range (min … max):   139.6 ms … 153.2 ms    20 runs

Summary
  typed_json ran
    1.10 ± 0.04 times faster than serde_json
```

### Release

```sh
$ hyperfine \
    --command-name "typed_json" \
    "pushd tests/crates/stress3 && touch src/main.rs && cargo build --release" \
    --command-name "serde_json" \
    "pushd tests/crates/stress4 && touch src/main.rs && cargo build --release"

Benchmark 1: typed_json
  Time (mean ± σ):     632.7 ms ±  10.2 ms    [User: 961.5 ms, System: 70.2 ms]
  Range (min … max):   610.5 ms … 647.2 ms    10 runs

Benchmark 2: serde_json
  Time (mean ± σ):      1.001 s ±  0.020 s    [User: 1.188 s, System: 0.077 s]
  Range (min … max):    0.973 s …  1.029 s    10 runs

Summary
  typed_json ran
    1.58 ± 0.04 times faster than serde_json
```

## One off large document

In this test, I have included the single JSON file in verbatim.
I don't think this is a realistic use case but still interesting

### Debug

```sh
$ hyperfine \
    --command-name "typed_json" \
    "pushd tests/crates/stress1 && touch src/main.rs && cargo build" \
    --command-name "serde_json" \
    "pushd tests/crates/stress2 && touch src/main.rs && cargo build"

Benchmark 1: typed_json
  Time (mean ± σ):     137.7 ms ±   4.6 ms    [User: 134.5 ms, System: 74.9 ms]
  Range (min … max):   128.1 ms … 146.1 ms    21 runs

Benchmark 2: serde_json
  Time (mean ± σ):     148.5 ms ±   6.8 ms    [User: 133.5 ms, System: 101.3 ms]
  Range (min … max):   139.3 ms … 164.6 ms    17 runs

Summary
  typed_json ran
    1.08 ± 0.06 times faster than serde_json
```

### Release

```sh
$ hyperfine \
    --command-name "typed_json" \
    "pushd tests/crates/stress1 && touch src/main.rs && cargo build --release" \
    --command-name "serde_json" \
    "pushd tests/crates/stress2 && touch src/main.rs && cargo build --release"

Benchmark 1: typed_json
  Time (mean ± σ):      2.616 s ±  0.014 s    [User: 3.932 s, System: 0.118 s]
  Range (min … max):    2.588 s …  2.638 s    10 runs

Benchmark 2: serde_json
  Time (mean ± σ):      1.281 s ±  0.014 s    [User: 1.554 s, System: 0.088 s]
  Range (min … max):    1.268 s …  1.305 s    10 runs

Summary
  serde_json ran
    2.04 ± 0.02 times faster than typed_json
```

## Conclusion

I don't think I can conclusively say that typed-json introduces a compile time regression in standard use.
At the extremes it likely will need to compile many more types but in standard use it can re-use a lot of prior compilations.
