# Release Process and Coding Standards

This page covers the hosted 2026.9 release process in
[`astrid-runtime/astrid`](https://github.com/astrid-runtime/astrid).
It was checked against runtime commit
[`28034075`](https://github.com/astrid-runtime/astrid/tree/280340757cb67478937184e352c286bb4384a6e5).
For later releases, inspect the workflows at the selected source commit.

## Version and changelog policy

Keep version changes in a dedicated release PR, not a feature or fix PR.
Use a conventional public title such as `feat(release): prepare 2026.9.0`.
The release PR updates the workspace version, lockfile and relevant release
metadata together; it is not necessarily a two-file change.

Astrid uses the calendar-based policy in
[release/VERSIONING.md](https://github.com/astrid-runtime/astrid/blob/main/release/VERSIONING.md).
The September 2026 release is `2026.9.0`, tagged `v2026.9.0`.
SDK and capsule versions are independent; do not infer them from the runtime.

Feature and fix PRs add `changes/{issue}.{kind}.md` fragments.
The supported kinds are `added`, `changed`, `deprecated`, `removed`,
`fixed`, and `security`. Documentation/CI-only exceptions follow the
repository's changelog policy. Do not add a nonstandard `Breaking` heading:
describe compatibility changes under the appropriate Keep a Changelog category.

The release PR uses `scripts/changelog.py roll` (see its `--help`) to
accumulate fragments. Review the result against the **previous public release**:
combine intermediate fixes to features that never shipped, preserve distinct
user-visible changes, and keep migration or compatibility warnings. The
changelog is an itemized account of shipped differences, not a transcript of
every review iteration. Preserve version headings, dates, and comparison links.

## Release execution

Tagging and publication require release authorization; a green PR alone is not
authorization. Re-verify the selected landed commit, version, changelog, and
required checks, then create a signed annotated tag for that exact commit.
Do not tag a local branch tip merely because its version is correct.

The executable procedure is
[release.yml](https://github.com/astrid-runtime/astrid/blob/main/.github/workflows/release.yml)
and the [release-channel guide](https://github.com/astrid-runtime/astrid/blob/main/docs/release-channels.md).
The 2026.9 publication matrix contains six Unix targets:

| Platform | Targets |
|---|---|
| macOS | `aarch64-apple-darwin`, `x86_64-apple-darwin` |
| Linux GNU | `aarch64-unknown-linux-gnu`, `x86_64-unknown-linux-gnu` |
| Linux MUSL | `aarch64-unknown-linux-musl`, `x86_64-unknown-linux-musl` |

Windows CI does not imply a Windows release archive. The package manifest and
archive validators, not a manually maintained file count, define archive
membership. Packages include the runtime tools and target-appropriate native
filesystem support.

Darwin packaging signs the filesystem companion and app and notarizes/staples
the app. The supervised FSKit gate binds an operator's local mount/IO evidence
to the same-run archive digest, source, run and attempt. It is **operator-attested
local execution**, not a claim that a hosted runner performed the mount.
Follow the checked-in certification scripts and protected approval procedure;
do not approve it merely because signing succeeded. Dedicated native-runner
certification remains a separate execution vehicle.

The publication path verifies its manifests and artifacts and signs release
metadata/assets. A local rehearsal proves behavior for the rehearsed bytes and
trust identity; future production digests and signing success are established
only by actual release execution. Failed required verification stops the
affected publication or downstream promotion.

Do not bypass channel gates with `cargo publish --workspace`. Inspect the
authorized promotion workflow and `scripts/publish_crates_io.sh` for the
publishable dependency closure and order. A Cargo installation is not equivalent
to the signed native-app archive.

## Runtime layout is not release packaging

Immutable executable files live outside the mutable runtime root. Current
durable state is held in `astrid.volume`; working projections exist while the
runtime runs and are retired on clean stop. Older descriptions of
`~/.astrid/bin/<hash>.wasm` and `meta.json` are not the current durable
load authority. See the
[Book's 2026.9 operating guide](https://github.com/astrid-runtime/book/blob/main/src/operating-2026-9.md).

## CI and pre-submission checks

Use the selected branch's `.github/workflows/ci.yml` and PR checks as the
authority for triggered jobs and supported targets; job counts change.

Typical Rust checks are:

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-features -- -D warnings
cargo test --workspace -- --quiet
```

Run focused regression and packaging/contract tests for the changed path too.
Report exact commands, outcomes, and any unexecuted platform check. A parsed
workflow or a passing textual contract test does not establish that an embedded
shell or Python program executes; exercise the program where practical.

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
- **`arithmetic_side_effects = "deny"`**: Clippy rejects the arithmetic patterns covered by this lint; this is not a proof that all arithmetic is overflow-free. Use checked arithmetic (`checked_add`, `saturating_mul`, etc.) or explicit casting when truncation is intentional.

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

The workspace-level `deny` means `unsafe` is forbidden by default. Examples of exceptions requiring a documented safety argument include:

- **Integration tests calling `std::env::set_var`** (Rust 2024 edition made it `unsafe`). These carry a `// Safety:` comment explaining the single-threaded invocation pattern. See `crates/astrid-integration-tests/tests/gateway_e2e.rs:59`.
- **Tests calling `std::env::remove_var`** for sandbox policy probing. See `crates/astrid-workspace/src/sandbox/mod.rs:755`.

These examples are not an exhaustive inventory of unsafe code. To introduce `unsafe` in production code, you need a Maintainer review, a detailed soundness argument in the code comment, and a linked issue tracking the technical debt.

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

The current file-size gate uses a 1000-line source cap and a 2000-line cap for recognized test files. Check its path classification and base-comparison rules in `pr-checks.yml`; do not assume any filename containing “test” qualifies. Split coherent modules rather than requesting an automatic waiver.
When a file approaches the limit, split it into a submodule directory. The `crates/astrid-capsule/src/manifest/` split (from a 1000-line single file into `mod.rs`, `capabilities.rs`, and `topics.rs`) is the canonical example.

---

## PR Requirements

All PRs must:

1. Be linked to an existing issue via `Closes #N` in the body. The `linked-issue` CI job enforces this. If no issue exists, open one first. No unsolicited PRs.
2. Fill the current template: **Linked Issue**, **Summary**, **Changes**, **Verification**, and **AI / Tool Assistance**, plus its checklist. The `template` job in `pr-checks.yml` rejects PRs with empty sections.
3. Add the required changelog fragment (or use the documented docs/CI exception); release PRs roll fragments.
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

**Do not open a public GitHub issue for security vulnerabilities.** Use [GitHub's private vulnerability reporting](https://github.com/astrid-runtime/astrid/security/advisories/new) to submit a report. This keeps the issue private until a fix is ready.

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
