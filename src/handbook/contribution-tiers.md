# Contribution Tiers and Security-Critical Crates

Astrid is a security-critical runtime. Every change that lands in `core/` is reviewed against its threat model, not just its correctness. The project uses a four-tier contributor system enforced by CI to protect the security boundary while keeping the door open for new contributors who follow the process.

## The Four Tiers

The canonical source is `.github/contributors.yml` (`core/.github/contributors.yml`). Anyone not listed there is treated as **new** by default. CI reads this file directly on every PR.

| Tier | Entry point | Scope |
|------|-------------|-------|
| **New** | Default for all unlisted contributors | Must open an issue first, wait for maintainer assignment, and have a maintainer add the `newcomer-approved` label before CI proceeds |
| **Astrinaut** | Promoted after a successful first contribution | Can self-claim issues and submit PRs to non-core crates: CLI, SDK, capsules, docs, tests |
| **Core** | Promoted after sustained quality contributions | Can modify core crates (kernel, events, hooks, config). Security-critical paths still require maintainer co-review |
| **Maintainer** | Project leads | Full access: security paths, refactors, releases, version bumps |

Promotions happen at maintainer discretion. There is no application process. The quality and consistency of your merged work is the signal.

The `contributors.yml` structure:

```yaml
maintainers:
  - username: joshuajbouw
    since: 2024-01-01

core: []

astrinauts: []
```

## How CI Enforces the Tiers

The `contributor-gate` job in `.github/workflows/pr-checks.yml` (lines 50-147) runs on every PR against `main`. It:

1. Parses `contributors.yml` to determine the PR author's tier.
2. Fetches the list of changed files from the GitHub API.
3. Tests each changed path against two path sets defined in the script:

```bash
SECURITY_PATHS="crates/astrid-crypto crates/astrid-capabilities crates/astrid-audit \
  crates/astrid-approval crates/astrid-vfs crates/astrid-storage crates/astrid-sys \
  crates/astrid-core"

CORE_PATHS="crates/astrid-kernel crates/astrid-events crates/astrid-hooks \
  crates/astrid-config crates/astrid-mcp"
```

The gate logic per tier:

- **new**: CI fails unless the PR carries the `newcomer-approved` label from a maintainer.
- **astrinaut**: CI fails if the PR touches any path under `SECURITY_PATHS` or `CORE_PATHS`.
- **core**: CI passes, but emits a warning if any file under `SECURITY_PATHS` is modified. That warning signals to reviewers that a maintainer co-review is required before merge.
- **maintainer**: All checks pass unconditionally.

## Security-Critical Crates

These eight crates form the security boundary. Only core-tier and maintainer-tier contributors can modify them. The `contributor-gate` CI job enforces this at the path level.

### `astrid-crypto` (`crates/astrid-crypto`)

Provides ed25519 key pairs, signatures, and BLAKE3 content hashing. Every capability token and audit entry is signed by this layer. The crate carries `#![deny(unsafe_code)]`, `#![deny(missing_docs)]`, `#![deny(clippy::all)]`, `#![deny(clippy::unwrap_used)]`, and `#![deny(unreachable_pub)]` in `src/lib.rs` (line 33-38).

Public surface: `KeyPair`, `PublicKey`, `Signature`, `ContentHash`.

### `astrid-capabilities` (`crates/astrid-capabilities`)

Capability tokens with ed25519 signatures, resource patterns with glob matching, session and persistent token storage, and per-principal authorization checks. Tokens are:

- Signed by the runtime's ed25519 key.
- Linked to the approval audit entry that created them.
- Time-bounded (optional expiration).
- Scoped (session or persistent).

Carries the same five crate-level deny attributes as `astrid-crypto` (`src/lib.rs`, lines 53-58).

### `astrid-audit` (`crates/astrid-audit`)

Chain-linked cryptographic audit logging. Each entry is signed by the runtime key and contains the BLAKE3 hash of the previous entry, providing tamper evidence. Any modification to historical entries breaks the chain and is detectable by `AuditLog::verify_chain`. Backed by SurrealKV for persistence. Same deny attributes at `src/lib.rs` lines 53-58.

### `astrid-approval` (`crates/astrid-approval`)

Human-in-the-loop approval gates for sensitive agent operations. Contains the `ApprovalManager`, allowance patterns, session and per-action budget tracking, a security policy (hard-blocked and approval-required tool lists), and a security interceptor combining all layers with intersection semantics. Same deny attributes at `src/lib.rs` lines 44-49.

### `astrid-vfs` (`crates/astrid-vfs`)

Virtual filesystem abstraction providing sandboxed operations, capability-based access, and a copy-on-write overlay implementation. The `boundary` and `worktree` submodules are `pub(crate)` only. Note: `OverlayVfs::commit` and `OverlayVfs::rollback` have no production caller as of this writing. Same deny attributes at `src/lib.rs` lines 6-11.

### `astrid-storage` (`crates/astrid-storage`)

Unified persistence layer with two tiers: raw key-value via SurrealKV (`KvStore`, enabled by the `kv` feature) and full SurrealQL via SurrealDB (`Database`, enabled by the `db` feature). Backs all system stores: approval, audit, capabilities, memory. The `secret` module manages keychain and secret store implementations. Same deny attributes at `src/lib.rs` lines 38-43.

### `astrid-sys` (`crates/astrid-sys`)

OS microkernel bindings referenced in `CONTRIBUTING.md`. This crate is listed in the `SECURITY_PATHS` enforcement set but lives in `sdk-rust/` (the standalone SDK), not under `core/crates/`. PRs against the SDK that touch `astrid-sys` fall outside the core `contributor-gate` job; capsule authors and SDK contributors should treat it as security-critical regardless.

### `astrid-core` (`crates/astrid-core`)

Foundation types and authorization interfaces used throughout the runtime: `PrincipalId`, `SessionId`, `Permission`, `Group`, capability grammar (`capability_matches`, `validate_capability`), elicitation primitives, uplink types, retry configuration, and profile/config types. Because this crate is a dependency of almost every other crate in the workspace, changes to it can silently affect the entire security model. Same deny attributes at `src/lib.rs` lines 10-15.

## What Distinguishes Security-Critical from Core

The `CORE_PATHS` set covers crates that are important but not themselves on the cryptographic or authorization boundary:

- `astrid-kernel`: runtime dispatch and the Wasmtime sandbox host.
- `astrid-events`: IPC event types and the bus.
- `astrid-hooks`: lifecycle hook execution.
- `astrid-config`: daemon and capsule configuration parsing.
- `astrid-mcp`: MCP protocol bridge.

Core-tier contributors can touch these without maintainer co-review. Modifications that approach the security boundary (for example, adding a new host function in `astrid-kernel` that calls into `astrid-capabilities`) should be discussed in the issue before the PR is opened.

## PR Requirements

All PRs against `core/` must pass five CI jobs defined in `.github/workflows/pr-checks.yml`:

**`template`**: Validates that the PR body fills in every required section of `.github/pull_request_template.md` (Linked Issue, Summary, Changes, Test Plan) and contains a real issue number. Empty or placeholder sections fail CI.

**`contributor-gate`**: Tier enforcement described above.

**`file-size`**: Rejects any file pushed over 1000 lines by the PR, measured against the base commit. Files already over 1000 lines on the base branch are noted but not blocked. The `large-file-ok` label (maintainer-only in practice) bypasses this check.

**`linked-issue`**: Requires a `Closes #N` reference in the PR body, or an issue linked via the GitHub sidebar. The check uses both regex on the PR body and the GraphQL closing-issues API.

**`account-age`**: Runs for brand-new GitHub accounts (NONE/FIRST_TIMER/FIRST_TIME_CONTRIBUTOR association). Emits warnings if the account is fewer than 30 days old or has no public repos and no followers.

Separately, `.github/workflows/changelog.yml` requires a `CHANGELOG.md` entry under `[Unreleased]` on every PR that changes `.rs` files or `Cargo.toml`. The `skip-changelog` label bypasses it.

The full CI suite (`.github/workflows/ci.yml`) runs `cargo check`, `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test --workspace --exclude astrid-openclaw` (on both Ubuntu and macOS), MSRV check at 1.95, and `cargo audit` via rustsec.

## Adversarial Self-Review

Before requesting review on any non-trivial change, work through these questions inline.

**Failure modes under production conditions.** Poisoned locks, killed processes, exhausted memory, interrupted syscalls, corrupted state across fork boundaries. Ask: how does this fail at 3am?

**Invariants of the execution context.** WASM guests cannot escape the sandbox. Untrusted input cannot reach `format!()` unsanitized. Capsule manifests are untrusted input; operator-only fields need `#[serde(skip_deserializing)]` and a parser-isolation test. Identify what cannot happen in your context, then check whether your change makes a violation possible.

**Boundary crossings.** If your change adds or modifies a host function, a capability check, an IPC topic, or an audit action, those are boundary crossings. Flag them explicitly in the PR summary. Do not bury them in unrelated diffs.

**Kernel purity.** The kernel is dumb. It routes events, enforces capabilities, and manages the WASM sandbox. It contains no business logic. If a review comment says "this belongs in a capsule," it is correct.

Adversarial review is not a checklist to be completed. It is a mindset: read your own diff as an attacker would.

## What Will Not Be Accepted

These are hard stops, not suggestions:

- PRs with no linked issue and no prior discussion.
- AI-generated bulk submissions that show no understanding of the affected crates.
- Refactors submitted by contributors below maintainer tier. If you see something that needs refactoring, open an issue.
- Changes to security-critical crates from contributors below core tier.
- `unsafe` code without explicit written justification in the PR and maintainer approval. Every security-critical crate carries `#![deny(unsafe_code)]` in its `lib.rs`.
- Version bumps mixed into feature or fix PRs. Version bumps are always a separate PR.

## Reporting Vulnerabilities

Do not open a public issue. Use [GitHub Security Advisories](https://github.com/unicity-astrid/astrid/security/advisories/new) to report privately. The project targets acknowledgment within 48 hours and a fix timeline within 7 days. See `core/SECURITY.md` for the full scope definition, which covers sandbox escapes, capability token forgery, cryptographic weaknesses in ed25519 or BLAKE3, SSRF or injection through host functions, and audit log tampering.

## See also

- [Release Process and Coding Standards](release-and-standards.md)
- [The RFC Trigger](rfc-trigger.md)
