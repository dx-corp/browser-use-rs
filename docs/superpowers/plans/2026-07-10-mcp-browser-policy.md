# MCP Browser Policy Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Carry explicit `BrowserProfile` policy through the four non-agent MCP tools and preserve that policy across persistent-session reconnects without storing profile secrets.

**Architecture:** MCP inputs gain an optional `browser_profile`. One-shot launches use it directly. Persistent session records store only a SHA-256 fingerprint of the canonical serialized profile; reconnects require the caller to supply the same profile and use `connect_with_profile`, preventing a restart from silently dropping URL policy while retaining backward compatibility for legacy unprofiled sessions.

**Tech Stack:** Rust 1.85+, serde/schemars, Tokio, CDP browser sessions, JSON-RPC MCP 2025-06-18.

## Global Constraints

- Keep `browser_use_agent` unchanged and provider secrets out of MCP tool schemas.
- Omitted `browser_profile` preserves current one-shot and legacy-session behavior.
- Never persist the serialized profile; persist only its SHA-256 fingerprint.
- A profiled session cannot be reused or reconnected without the identical canonical profile.
- Use TDD: every production behavior starts with a focused failing test.

---

### Task 1: Add profile-bearing MCP contracts

**Files:**
- Modify: `crates/browser-use-mcp/src/lib.rs`

**Interfaces:**
- Consumes: `browser_use_cdp::BrowserProfile`.
- Produces: `browser_profile: Option<BrowserProfile>` on `StateToolInput`, `ActionsToolInput`, `ReplayToolInput`, and `SessionToolInput`.

- [ ] **Step 1: Write failing schema and serde tests**

Add tests that deserialize `{"url":"data:text/html,ok","browser_profile":{"allowed_domains":["example.com"],"block_ip_addresses":true}}`, assert the profile fields, and assert each selected tool's generated `inputSchema` contains `browser_profile`. Assert the agent schema still does not gain that field.

- [ ] **Step 2: Verify RED**

Run: `cargo test --locked -p browser-use-mcp browser_profile -- --nocapture`

Expected: compilation or assertion failure because the four input structs lack `browser_profile`.

- [ ] **Step 3: Add the minimal fields**

Add this field to each selected input struct:

```rust
/// Browser launch and URL policy applied to a new or reconnected session.
#[serde(default, skip_serializing_if = "Option::is_none")]
pub browser_profile: Option<BrowserProfile>,
```

Import `BrowserProfile` from `browser_use_cdp` and add the crate dependency if it is not already direct.

- [ ] **Step 4: Verify GREEN**

Run: `cargo test --locked -p browser-use-mcp browser_profile -- --nocapture`

Expected: all focused tests pass.

- [ ] **Step 5: Commit**

```bash
git add Cargo.lock crates/browser-use-mcp
git commit -m "feat(mcp): accept browser profiles on browser tools"
```

### Task 2: Fingerprint persistent-session profiles

**Files:**
- Modify: `crates/browser-use-mcp/src/lib.rs`
- Modify: `crates/browser-use-cli/src/session.rs`
- Modify: `crates/browser-use-cli/src/tests.rs`

**Interfaces:**
- Produces: `SessionRecord.browser_profile_fingerprint: Option<String>`.
- Produces: `browser_profile_fingerprint(&BrowserProfile) -> anyhow::Result<String>`.
- Produces: `start_persistent_session_with_profile(id, url, screenshot, BrowserProfile)`.

- [ ] **Step 1: Write failing record and fingerprint tests**

Cover legacy record JSON without the field, deterministic equality for identical profiles, inequality after changing `allowed_domains`, and verify the fingerprint string does not contain a configured header or cloud API key.

- [ ] **Step 2: Verify RED**

Run: `cargo test --locked -p browser-use-cli browser_profile_fingerprint -- --nocapture`

Expected: failure because the record field and helper do not exist.

- [ ] **Step 3: Implement canonical fingerprinting and profiled start**

Use `serde_json::to_vec(profile)` followed by `Sha256::digest`, encode lowercase hex, and store only the digest:

```rust
pub(crate) fn browser_profile_fingerprint(profile: &BrowserProfile) -> anyhow::Result<String> {
    use sha2::{Digest, Sha256};
    let bytes = serde_json::to_vec(profile)?;
    Ok(format!("{:x}", Sha256::digest(bytes)))
}
```

Add the optional record field with serde defaults. Make the existing `start_persistent_session` call a new profiled helper using `BrowserProfile::default()`. The helper overrides `user_data_dir` with the session-owned directory, launches with that profile, connects with `connect_with_profile`, and records the original effective profile fingerprint.

- [ ] **Step 4: Verify GREEN**

Run: `cargo test --locked -p browser-use-cli browser_profile_fingerprint -- --nocapture`

Expected: focused tests pass.

- [ ] **Step 5: Commit**

```bash
git add crates/browser-use-mcp crates/browser-use-cli
git commit -m "feat(session): bind persistent sessions to browser policy"
```

### Task 3: Apply and enforce profiles in MCP execution

**Files:**
- Modify: `crates/browser-use-cli/src/mcp_daemon.rs`
- Modify: `crates/browser-use-cli/src/tests.rs`

**Interfaces:**
- Changes `McpRuntime::session(&mut self, session_id, url, browser_profile)`.
- Uses `launch_and_navigate_with_profile` and `CdpBrowserSession::connect_with_profile`.

- [ ] **Step 1: Write failing session-plan and request-routing tests**

Add pure tests covering: new profiled session accepted; in-memory profiled reuse with identical fingerprint accepted; mismatched or omitted profile rejected; persistent profiled reconnect with identical profile accepted; legacy unprofiled reconnect remains accepted only when no profile is supplied. Add handler tests proving each selected tool forwards the profile.

- [ ] **Step 2: Verify RED**

Run: `cargo test --locked -p browser-use-cli mcp_profile -- --nocapture`

Expected: failure because runtime session routing ignores profiles.

- [ ] **Step 3: Implement minimal enforcement**

Store an in-memory session entry containing both `Arc<CdpBrowserSession>` and `Option<String>` fingerprint. Compare expected and supplied fingerprints before navigation. Reconnect legacy records with `connect` only when both record and call are unprofiled; reconnect profiled records only with an identical supplied profile and `connect_with_profile`. Launch one-shot calls with the supplied profile or default. Route `SessionOperation::Start` through the profiled persistent-session helper.

- [ ] **Step 4: Verify GREEN**

Run: `cargo test --locked -p browser-use-cli mcp_profile -- --nocapture`

Expected: focused tests pass.

- [ ] **Step 5: Add a browser-backed deny test**

Launch a profile allowing only `example.test`, start on a data URL, then attempt a prohibited navigation and assert the action result contains the URL-policy failure without reaching the target.

- [ ] **Step 6: Run the package suites**

Run:

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
python3 -m unittest scripts/test_openai_mcp_smoke.py
```

Expected: exit 0 for every command.

- [ ] **Step 7: Commit**

```bash
git add crates/browser-use-cli docs/MCP.md
git commit -m "feat(mcp): enforce browser profiles across sessions"
```

