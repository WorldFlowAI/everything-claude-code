# Build and Fix

Incrementally fix build errors.

**Rust projects (a `Cargo.toml` exists):** delegate to the `rust-build-resolver` agent. It runs `cargo check --workspace --all-targets`, fixes errors in dependency order (lowest crate first), then verifies with clippy (`-D warnings`) and tests. The steps below describe the TypeScript/JS flow.

1. Run build: npm run build or pnpm build

2. Parse error output:
   - Group by file
   - Sort by severity

3. For each error:
   - Show error context (5 lines before/after)
   - Explain the issue
   - Propose fix
   - Apply fix
   - Re-run build
   - Verify error resolved

4. Stop if:
   - Fix introduces new errors
   - Same error persists after 3 attempts
   - User requests pause

5. Show summary:
   - Errors fixed
   - Errors remaining
   - New errors introduced

Fix one error at a time for safety!
