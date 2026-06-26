# Rust

Use idiomatic Rust error handling, ownership, and async runtime patterns.

## Error handling

Use `?` and `Result` types for recoverable errors. Avoid `.unwrap()` and `.expect()` in production code; keep them to examples, tests, or code paths where a panic is explicitly intended and justified.

Good:

```rust
fn read_config(path: &Path) -> Result<String, std::io::Error> {
    std::fs::read_to_string(path)
}
```

Bad:

```rust
fn read_config(path: &Path) -> String {
    std::fs::read_to_string(path).unwrap()
}
```

## String handling

Use `&str` for borrowed string slices and `String` for owned strings. Avoid unnecessary `.to_string()` and `.clone()` calls.

## Async runtime

Use current Tokio with `async`/`await`. Avoid older futures 0.1 APIs.
