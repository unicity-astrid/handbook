# Release Process and Coding Standards

This page documents the release workflow for the `core` repository (`unicity-astrid/astrid`), the lint and formatting rules every crate must satisfy, and the security disclosure process. Read it before opening your first PR against a kernel or security-critical crate.

---

## Version Bump Policy

Version bumps are **always a separate PR**. Never bundle a version bump with a feature or fix PR. The commit type is `chore: release X.Y.Z` and it touches only `Cargo.toml` workspace version and the `CHANGELOG.md` heading change from `[Unreleased]` to `[X.Y.Z] - YYYY-MM-DD`. Reviewers must be able to diff a feature PR and see only the intended change.

All workspace members inherit the version from the root `Cargo.toml`:

```toml
# core/Cargo.toml
[workspace.package]
version = "0.7.0"
```

Every crate in `[workspace.members]` carries `version.workspace = true`. Bumping the root version bumps every crate atomically. Do not set a crate-local version unless you have an explicit reason.

---

## Changelog Discipline

The project follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) with Semantic Versioning. Every PR that touches Rust source or `Cargo.toml` must include a `CHANGELOG.md` entry under `[Unreleased]`. The CI changelog enforcer (`dangoslen/changelog-enforcer`) rejects PRs that omit this step (`.github/workflows/changelog.yml`). PRs carrying the `skip-changelog` label are exempt, reserved for pure documentation and CI changes.

Section headings are `### Added`, `### Changed`, `### Fixed`, `### Removed`, `### Deprecated`, `### Security`, and `### Breaking`. Write entries in past tense, bold the feature name, and close with `Closes #N` where N is the linked issue. Keep entries dense but complete: reviewers use them to understand blast radius.

---

## Release Workflow

Releases are triggered by pushing a `v*` tag. The `.github/workflows/release.yml` pipeline runs automatically.

### Tagging and Triggering

```bash
# After the version-bump PR is merged to main:
git tag v0.8.0
git push origin v0.8.0
```

The pipeline fires on any tag matching `v[0-9]+.*`.

### Build Matrix

The release workflow builds four targets in parallel (`fail-fast: false`):

| Target | OS |
|--------|-----|
| `x86_64-apple-darwin` | `macos-latest` |
| `aarch64-apple-darwin` | `macos-latest` |
| `x86_64-unknown-linux-gnu` | `ubuntu-latest` |
| `aarch64-unknown-linux-gnu` | `ubuntu-latest` |

The Linux ARM target requires the `gcc-aarch64-linux-gnu` cross-compiler, installed by the workflow via `apt-get`. The `CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_LINKER` environment variable is set to `aarch64-linux-gnu-gcc` for that build only.

Only the `astrid` binary package is built for release: `cargo build --release --target ${{ matrix.target }} -p astrid`. The four companion binaries (`astrid`, `astrid-daemon`, `astrid-build`, `astrid-emit`) are copied from the target output directory, bundled into a `tar.gz` archive named `astrid-{VERSION}-{TARGET}.tar.gz`, and uploaded as build artifacts.

### Content-Addressed Binaries

At install time the kernel stores every capsule's WASM binary under a content-addressed path (`~/.astrid/bin/<blake3-hex>.wasm`) rather than by name. The install machinery in `crates/astrid-capsule-install/src/wasm.rs` hashes from the **source** file using BLAKE3, writes the result atomically via a UUID-suffixed temp file and `rename`, and never writes a `.wasm` into the per-capsule directory:

```rust
// crates/astrid-capsule-install/src/wasm.rs
let hash = blake3::hash(&bytes).to_hex().to_string();
let store_path = bin_dir.join(format!("{hash}.wasm"));
if !store_path.exists() {
    let tmp = bin_dir.join(format!("{hash}.tmp.{}", uuid::Uuid::new_v4().simple()));
    std::fs::write(&tmp, &bytes)?;
    std::fs::rename(&tmp, &store_path)?;
}
```

The UUID-suffix is required because the gateway processes admin requests concurrently (since the bus-direct refactor), and sibling tokio tasks in the same daemon process share a PID. A UUID is the only safe disambiguator.

The runtime resolves a capsule's WASM via `resolve_content_addressed_wasm` in `crates/astrid-capsule/src/engine/wasm/mod.rs` (line 90): it reads `meta.json` in the per-capsule directory to find `wasm_hash`, then constructs the path `$ASTRID_HOME/bin/<hash>.wasm`. WIT files follow the same scheme under `~/.astrid/wit/`. The `AstridHome` directory layout is defined in `crates/astrid-core/src/dirs.rs`; `bin_dir()` returns `{root}/bin`, `wit_dir()` returns `{root}/wit`.

### GitHub Release

After all four build jobs complete, the `github-release` job:

1. Downloads all `binary-*` artifacts.
2. Extracts the changelog section for the version from `CHANGELOG.md` using `awk` and embeds it in the release body.
3. Computes SHA-256 checksums with `sha256sum *.tar.gz > SHA256SUMS.txt`.
4. Creates the GitHub release with `softprops/action-gh-release`, attaching all four archives and `SHA256SUMS.txt`.
5. Dispatches a `release` event to the `homebrew-tap` repository so the Homebrew formula can be updated automatically.

The release body includes install instructions for both `cargo install astrid` (requires Rust 1.95 or later) and the pre-built binary path.

---

## CI Pipeline

The standard CI pipeline (`.github/workflows/ci.yml`) runs six jobs on every push to `main` or a `stream-*` branch, and on every PR targeting `main`. All jobs pin to Rust 1.95.

| Job | Command | Notes |
|-----|---------|-------|
| `check` | `cargo check --workspace --all-features` | Ubuntu only |
| `fmt` | `cargo fmt --all -- --check` | Fails on any formatting divergence |
| `clippy` | `cargo clippy --workspace --all-features -- -D warnings` | All warnings are errors in CI |
| `test` | `cargo test --workspace --exclude astrid-openclaw` | Ubuntu and macOS; `astrid-openclaw` excluded (parked) |
| `msrv` | `cargo check --workspace` at 1.95 | Verifies the declared `rust-version` is accurate |
| `audit` | `rustsec/audit-check` | Scans `Cargo.lock` for known CVEs |

All jobs check out submodules recursively (`submodules: recursive`) because `astrid-capsule`'s `build.rs` stages WIT files from the `wit/` submodule before `bindgen` runs.

### Pre-Submission Checklist

Before pushing a branch:

```bash
cargo fmt --all
cargo clippy --workspace --all-features -- -D warnings
cargo test --workspace --exclude astrid-openclaw
```

The PR template (`pull_request_template.md`) requires all three to pass. CI enforces the same commands and rejects PRs with unchecked boxes or empty template sections.

---

## Lint and Safety Standards

### Workspace-Level Lints

Three workspace-wide lint rules are declared in `core/Cargo.toml` under `[workspace.lints]`:

```toml
[workspace.lints.rust]
unsafe_code = "deny"

[workspace.lints.clippy]
all = { level = "warn", priority = -1 }
pedantic = { level = "warn", priority = -1 }
arithmetic_side_effects = "deny"
```

Every crate in the workspace inherits these rules via `[lints] workspace = true` in its own `Cargo.toml`. The implications:

- **`unsafe_code = "deny"`**: No `unsafe` blocks anywhere in the workspace unless explicitly overridden with a documented justification. See the Unsafe Code section below.
- **`clippy::all` and `clippy::pedantic` at `warn`**: Every pedantic lint fires as a warning. CI promotes all warnings to errors via `-D warnings`, so in practice `pedantic` is `deny` in CI.
- **`arithmetic_side_effects = "deny"`**: Integer overflow, underflow, and wrapping arithmetic are compile-time errors. Use checked arithmetic (`checked_add`, `saturating_mul`, etc.) or explicit casting when truncation is intentional.

### Clippy Configuration

`core/clippy.toml` applies to the whole workspace:

```toml
msrv = "1.95"
cognitive-complexity-threshold = 25
too-many-arguments-threshold = 7
too-many-lines-threshold = 100
type-complexity-threshold = 250

disallowed-methods = [
    { path = "std::env::set_var", reason = "Use safe configuration instead" },
    { path = "std::env::remove_var", reason = "Use safe configuration instead" },
]
```

`std::env::set_var` and `std::env::remove_var` are banned because they mutate a process-global table that is not thread-safe in Rust's 2024 edition. Tests that genuinely need to vary environment variables must document a `// SAFETY:` comment explaining why no other thread can observe the mutation (see `crates/astrid-workspace/src/sandbox/mod.rs:755` for an example). Production code never calls these functions.

`doc-valid-idents` is set to recognize `Astrid`, `MCP`, `WASM`, `WASI`, `OAuth`, `GitHub`, `macOS`, and `WebAssembly` so clippy does not flag these as typos in doc comments.

### Formatting

`core/rustfmt.toml` is the canonical formatter configuration:

```toml
edition = "2024"
max_width = 100
tab_spaces = 4
newline_style = "Unix"
reorder_imports = true
match_block_trailing_comma = true
use_field_init_shorthand = true
use_try_shorthand = true
```

CI runs `cargo fmt --all -- --check` and fails on any deviation. Format before committing: `cargo fmt --all`.

### Unsafe Code

The workspace-level `deny` means `unsafe` is forbidden by default. The only legitimate exceptions in the codebase:

- **Integration tests calling `std::env::set_var`** (Rust 2024 edition made it `unsafe`). These carry a `// Safety:` comment explaining the single-threaded invocation pattern. See `crates/astrid-integration-tests/tests/gateway_e2e.rs:59`.
- **Tests calling `std::env::remove_var`** for sandbox policy probing. See `crates/astrid-workspace/src/sandbox/mod.rs:755`.

Neither case is production code. No production crate uses `unsafe`. To introduce `unsafe` in production code, you need a Maintainer review, a detailed soundness argument in the code comment, and a linked issue tracking the technical debt.

### Security-Critical Crate Attributes

Beyond the workspace defaults, the seven security-critical crates add crate-level deny attributes in their `lib.rs`:

```rust
#![deny(unsafe_code)]
#![deny(missing_docs)]
#![deny(clippy::all)]
#![deny(unreachable_pub)]
#![deny(clippy::unwrap_used)]
#![cfg_attr(test, allow(clippy::unwrap_used))]
```

The security-critical crates are: `astrid-crypto`, `astrid-capabilities`, `astrid-audit`, `astrid-approval`, `astrid-vfs`, `astrid-storage`, and `astrid-core`. Only Maintainer-tier contributors may modify these crates; Core-tier contributors touching them cause a CI warning requesting maintainer co-review, but the check is not blocking (see `.github/workflows/pr-checks.yml`).

`#![deny(missing_docs)]` means every public item must have a doc comment. `#![deny(unreachable_pub)]` means every `pub` item must actually be reachable from the crate root. Together they push the public API surface to be both documented and intentional.

`#![deny(clippy::unwrap_used)]` bans `.unwrap()` in production code. Use `?`, `expect("reason")` when the invariant is local and obvious, or explicit pattern matching. Test code relaxes this via the `cfg_attr` line.

### Doc Comment Style

Doc comments must not contain em-dashes. Use periods, commas, parentheses, or the word "and" instead. This applies to all `///` and `//!` comments throughout the codebase.

Example of correct style:

```rust
/// Resolve the home directory.
///
/// Checks `$ASTRID_HOME` first, then falls back to `$HOME/.astrid/`.
///
/// Returns an error if neither `$ASTRID_HOME` nor `$HOME` is set.
pub fn resolve() -> io::Result<Self> {
```

### File Size Limit

Individual source files must not exceed 1000 lines. The `pr-checks.yml` workflow measures every file changed by a PR against its line count on the base branch and fails if any file crosses 1000 lines that was under the limit before the PR. The `large-file-ok` label overrides this check and is available only to Maintainers for refactors where the boundary is difficult to draw incrementally.

When a file approaches the limit, split it into a submodule directory. The `crates/astrid-capsule/src/manifest/` split (from a 1000-line single file into `mod.rs`, `capabilities.rs`, and `topics.rs`) is the canonical example.

---

## PR Requirements

All PRs must:

1. Be linked to an existing issue via `Closes #N` in the body. The `linked-issue` CI job enforces this. If no issue exists, open one first. No unsolicited PRs.
2. Have all four template sections filled: **Linked Issue**, **Summary**, **Changes**, **Test Plan**. The `template` job in `pr-checks.yml` rejects PRs with empty sections.
3. Update `CHANGELOG.md` under `[Unreleased]`. The changelog enforcer fails the PR otherwise.
4. Pass `cargo test --workspace` and `cargo clippy -- -D warnings`.

New contributors additionally require a maintainer to add the `newcomer-approved` label before CI proceeds on their PR.

---

## Toolchain

```toml
# core/rust-toolchain.toml
[toolchain]
channel = "1.95.0"
components = ["rustfmt", "clippy"]
```

The workspace pins to Rust 1.95.0. The MSRV is set to match: `rust-version = "1.95"` in `[workspace.package]`. Do not use stabilized features from later versions. The CI MSRV job (`cargo check --workspace` at 1.95) verifies this on every PR.

The Rust 2024 edition is in use (`edition = "2024"` in `[workspace.package]` and in `rustfmt.toml`).

---

## Security Disclosure

### Reporting a Vulnerability

**Do not open a public GitHub issue for security vulnerabilities.** Use [GitHub's private vulnerability reporting](https://github.com/unicity-astrid/astrid/security/advisories/new) to submit a report. This keeps the issue private until a fix is ready.

Include in your report:

- A description of the vulnerability.
- Steps to reproduce.
- The affected components (crate name and module).
- A severity assessment if you have one.

The project aims to acknowledge reports within 48 hours and provide a fix timeline within 7 days.

### In-Scope Vulnerabilities

`core/SECURITY.md` defines scope explicitly. In scope:

- Sandbox escapes: a WASM guest accessing host resources outside its granted capabilities.
- Capability token forgery or privilege escalation.
- Cryptographic weaknesses in ed25519 signing, BLAKE3 verification, or the audit chain.
- SSRF or injection through capsule host functions.
- Audit log tampering or bypass.

Out of scope:

- Denial of service through resource exhaustion (covered by the capability limits and quota system).
- Vulnerabilities in upstream dependencies (report to the upstream project).
- Issues requiring physical access to the host machine.

### Threat Model Questions

Apply these questions to every non-trivial change before requesting review. They are the adversarial self-review gate:

1. What is the threat model?
2. How can this be abused?
3. What cryptographic proof exists?
4. Is this auditable?
5. Does it fail secure?

The fail-secure invariant is non-negotiable. The sandbox fails closed. The capability store degrades to "always ask" rather than "always allow". Audit logging continues and alerts rather than silently dropping entries. A change that introduces a soft fallback on a security boundary requires explicit Maintainer sign-off with a written rationale.

---

## Build Provenance Metric

The gateway crate captures build provenance at compile time via `crates/astrid-gateway/build.rs`. It runs `git rev-parse --short=12 HEAD` and the `rustc --version` of the invoking toolchain and exposes them as `ASTRID_GIT_SHA` and `ASTRID_RUSTC_VERSION` environment variables, which `src/metrics.rs` reads via `env!`. Both fall back to `"unknown"` when the `.git` directory is unreachable (source-tarball builds), so `env!` never fails.

The metric `astrid_build_info{version, git_sha, rustc}` is a gauge pinned at `1.0`, exposed on the unauthenticated `GET /metrics` endpoint. Its value is the label set. Dashboards use it for build-provenance joins:

```promql
astrid_build_info{git_sha="abc123def456"}
```

This metric is intentionally unauthenticated: it carries no principal, path, or secret, and its value is the same for every request to the same daemon process.

## See also

- [Contribution Tiers and Security-Critical Crates](contribution-tiers.md)
