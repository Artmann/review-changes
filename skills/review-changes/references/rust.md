# Rust Guidelines

- `unwrap()` on `Result`/`Option` in non-test code — use `?` operator or handle the error case
- Missing error propagation — prefer `?` operator over manual `match` on `Result`
- `unsafe` blocks without justification comment explaining why it's safe
- Unnecessary `.clone()` calls — prefer borrowing when ownership isn't needed
