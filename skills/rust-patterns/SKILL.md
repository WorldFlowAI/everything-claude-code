---
name: rust-patterns
description: Rust idioms and project conventions for writing, testing, reviewing, and securing Rust code in cargo projects and workspaces. Use whenever the project has a Cargo.toml and you are planning, implementing, reviewing, or testing Rust code: error handling (thiserror/anyhow), ownership, async/tokio, workspace and crate layout, testing (proptest, insta, mockall), performance, unsafe, and handling secrets/LLM data.
---

# Rust Patterns

Conventions for Rust work in this toolkit. They override the TypeScript-oriented rules in `coding-standards`, `tdd-workflow`, and `rules/` when the code is Rust (for example, console.log checks become `println!`/`dbg!` checks, and immutability patterns are expressed through ownership rather than spread operators).

**Always defer to the project first**: `CLAUDE.md`, `clippy.toml`, `rustfmt.toml`, `deny.toml`, CI workflows, and patterns already used in the codebase win over anything here.

## Commands (the Rust verification loop)

```bash
cargo fmt --all -- --check
cargo check --workspace --all-targets --all-features
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --workspace --all-features          # or: cargo nextest run --workspace
cargo test --doc --workspace                   # nextest does not run doctests
cargo audit                                    # known vulnerabilities (cargo install cargo-audit)
cargo deny check                               # licenses, bans, advisories, sources (if deny.toml exists)
cargo llvm-cov --workspace --summary-only      # coverage (cargo install cargo-llvm-cov)
```

`verification-loop` and `/verify` use these instead of npm commands when a `Cargo.toml` is present. Debris to flag instead of `console.log`: `println!`, `eprintln!`, `dbg!`, `todo!`, `unimplemented!` outside tests, and new `.unwrap()` in library code.

## Error Handling

| Crate type | Use | Why |
|-----------|-----|-----|
| Library crate | `thiserror` enum per module or crate | Callers can match on variants |
| Binary / application top level | `anyhow::Result` with `.context()` | Rich error chains, no boilerplate |

```rust
// Library
#[derive(Debug, thiserror::Error)]
pub enum StoreError {
    #[error("record {id} not found")]
    NotFound { id: String },
    #[error("database error")]
    Db(#[from] sqlx::Error),
}

// Application
use anyhow::Context;
let cfg = std::fs::read_to_string(&path)
    .with_context(|| format!("reading config from {}", path.display()))?;
```

- Propagate with `?`. Use `.unwrap()` / `.expect("why this can't fail")` only for true invariants, tests, and startup code.
- Never silently discard a `Result` (`let _ = fallible();`) without a comment explaining why.
- Don't put secrets or full request bodies into error messages.
- Map errors at layer boundaries (DB error → domain error → HTTP status) instead of leaking low-level types upward.

## Ownership and API Design

- Accept the most general borrowed form: `&str` over `&String`, `&[T]` over `&Vec<T>`, `impl AsRef<Path>` for paths, `impl Into<String>` for owned-string constructors.
- Return owned types from constructors and builders. Return references only when tied to `&self`.
- Prefer `Option`/`Result` combinators (`map`, `and_then`, `ok_or_else`) and `let-else` over nested `match`.
- Newtypes for domain IDs and validated values (`struct UserId(Uuid)`, `struct Email(String)` with a fallible constructor). Make invalid states unrepresentable.
- Keep `pub` surface minimal: `pub(crate)` by default. Everything `pub` in a library crate is API you must maintain.
- Derive `Debug` everywhere **except** on types holding secrets (see Security).
- `Arc<T>` for shared read-only state; `Arc<Mutex<T>>` / `Arc<RwLock<T>>` only for shared mutation. Prefer message passing (channels) or ownership transfer when it's natural.
- Use `Cow<'_, str>` only when profiling shows allocation matters.

## Async (tokio)

- One runtime, created at the binary boundary (`#[tokio::main]`). Libraries never create runtimes.
- **Never block the runtime:** use `spawn_blocking` for CPU-heavy work or sync I/O, and `tokio::fs` / async drivers for I/O.
- **Never hold a `std::sync::Mutex` guard across `.await`.** Scope the guard. Use `tokio::sync::Mutex` only if the lock must span an await.
- **Cancellation safety:** any future in `tokio::select!` can be dropped at an await point. Don't keep partial state that would be lost. Document cancel-safety on public async fns used in `select!`.
- Bound concurrency: `Semaphore`, `buffer_unordered(n)`, or `JoinSet` with limits, never unbounded `spawn` per item.
- Use timeouts on every network call (`tokio::time::timeout` or client-level timeouts), with retries and backoff for transient errors.
- Graceful shutdown: propagate a `CancellationToken` (tokio-util) or broadcast channel. Await `JoinHandle`s on shutdown.
- Tasks: `tokio::spawn` requires `Send + 'static`. Move owned data or `Arc`s in, and handle the `JoinHandle` result (panics surface there).

## Workspace and Crate Layout

```
Cargo.toml              # [workspace], [workspace.dependencies], [workspace.lints]
crates/
  core/                 # domain types + logic, no I/O, no async runtime dependency
  storage/              # DB/persistence, depends on core
  api/                  # HTTP/gRPC layer, depends on core + storage
  cli/ or server/       # binary: wiring, config, runtime, anyhow
```

- Dependencies point inward toward `core`. No cycles. A cycle means a type belongs in a lower crate.
- Centralize versions in `[workspace.dependencies]`. Members use `{ workspace = true }`.
- Centralize lints in `[workspace.lints]` (Rust ≥ 1.74). Members use `[lints] workspace = true`.
- Features are additive. Never use a feature to remove functionality. Every crate must build alone (`cargo check -p <crate>`) without relying on features enabled by siblings.
- Split a crate when compile times hurt or a clear boundary exists, not preemptively.

## Testing (TDD in Rust)

The `tdd-workflow` cycle (red → green → refactor) applies unchanged. The tools differ:

| Need | Tool |
|------|------|
| Unit tests | `#[cfg(test)] mod tests` in the same file (can test private items) |
| Integration tests | `tests/*.rs`, public API only |
| Async tests | `#[tokio::test]` (`flavor = "multi_thread"` when testing spawn/Send behavior) |
| Property tests | `proptest`: parsers, serializers, invariants, round-trips |
| Snapshot tests | `insta`: complex outputs, API responses, prompts. Review with `cargo insta review` |
| Mocking | Define a trait at the boundary, mock with `mockall` or a hand-written fake. Don't mock what you own when a real in-memory implementation is easy |
| HTTP mocking | `wiremock` |
| DB tests | `sqlx::test` or testcontainers. Each test gets an isolated DB/transaction |
| Doc examples | Doctests on public APIs. They are compiled and run by `cargo test` |
| Faster runner | `cargo nextest` (remember doctests separately) |

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use proptest::prelude::*;

    #[test]
    fn rejects_empty_email() {
        assert!(matches!(Email::parse(""), Err(EmailError::Empty)));
    }

    proptest! {
        #[test]
        fn roundtrip(s in "[a-z]{1,20}@[a-z]{1,10}\\.com") {
            let e = Email::parse(&s).unwrap();
            prop_assert_eq!(e.as_str(), s);
        }
    }
}
```

Coverage: `cargo llvm-cov`. Aim for the project's target (toolkit default 80%), focused on logic crates (`core`) rather than wiring.

## Performance

- Measure before optimizing: `criterion` benchmarks for hot paths, `cargo flamegraph` / `samply` for profiles.
- Avoid needless allocation in hot loops: reuse buffers (`clear()` instead of a new `Vec`), `with_capacity` when the size is known, iterate instead of `collect()`ing intermediates.
- Release profile for production builds; consider `lto = "thin"` and `codegen-units = 1` for binaries after measuring.
- Serialization: `serde` with `#[serde(borrow)]` for zero-copy where the profile shows it matters.
- Don't reach for `unsafe` for speed without a benchmark proving the gain and a review.

## Security

### unsafe
- Avoid it. If required, keep it in a small module with a safe wrapper API, and add a `// SAFETY:` comment on every `unsafe` block explaining which invariants hold.
- Consider `#![forbid(unsafe_code)]` in crates that don't need it.
- Run `cargo +nightly miri test` on crates containing unsafe if feasible.

### Secrets and sensitive data (critical for LLM workloads)
- Wrap API keys, tokens, and credentials in `secrecy::SecretString`. They won't print via `Debug` or `Display`, and you access them explicitly with `.expose_secret()`.
- Use `zeroize` for key material held in memory.
- **Never derive `Debug` on structs containing prompts, completions, user content, PII, or secrets.** Implement `Debug` manually with redaction. `tracing` and error messages commonly print `{:?}`.
- With `tracing`, use `#[instrument(skip(request, api_key))]` or `skip_all` and record only safe fields. Never log full prompts or model responses by default; gate verbose logging behind an explicit debug flag and redact PII.
- Validate and bound all external input: size limits on request bodies and on LLM inputs/outputs, and timeouts.
- Treat LLM output as untrusted: parse into typed structs (`serde` + validation), never pass it to shell commands, SQL, or file paths unvalidated.

### Dependencies
- `cargo audit` in CI. `cargo deny` for license policy, banned crates, and duplicate versions.
- Prefer `rustls` over OpenSSL when the crate supports it (fewer system dependencies, memory-safe).
- Review new dependencies: maintenance status, download count, `unsafe` usage (`cargo geiger`), transitive tree size.

### Integer and parsing safety
- Use `checked_*` / `saturating_*` arithmetic for values from external input. Release builds wrap on overflow silently.
- `TryFrom` over `as` for narrowing conversions.
- Never use floats for money. Use integer minor units or `rust_decimal`.

## Code Review Checklist (Rust)

For use by `code-reviewer` and `security-reviewer`:

- [ ] No new `.unwrap()`/`.expect()` in library code without a justified invariant
- [ ] No `unsafe` without a `// SAFETY:` comment and a safe wrapper
- [ ] No locks held across `.await`, no blocking calls in async
- [ ] Errors carry context and don't leak secrets or PII
- [ ] Secret and PII types don't derive `Debug`; `tracing` spans skip sensitive args
- [ ] Public API changes are intentional and documented (semver)
- [ ] New dependencies are justified and pass `cargo audit` / `cargo deny`
- [ ] Clippy clean with `-D warnings`; no blanket `#[allow]`
- [ ] Tests cover new logic (unit + integration where boundaries change)
- [ ] No `println!`/`dbg!`/`todo!` left in non-test code

## Related

- `rust-build-resolver` agent: fixes failing cargo build, clippy, and test runs with minimal diffs
- `security-review` skill: general OWASP checklist (apply alongside the Rust security section above)
- `tdd-workflow` skill: the TDD process (use the Rust tooling above)
