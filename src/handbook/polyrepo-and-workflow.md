# Working on Astrid: The Polyrepo and Git Workflow

Astrid is not a monorepo. The kernel, the SDK, the RFCs, and every capsule each live in their own
GitHub repository under the `unicity-astrid` organization. Your local checkout layers all of these
as independent git repos under a single parent directory. Understanding that layout is the
prerequisite for every workflow decision that follows.

## The Polyrepo Map

Your working directory contains these top-level components, each a standalone git repo:

| Local path | GitHub repo | Current version | Purpose |
|---|---|---|---|
| `core/` | `unicity-astrid/astrid` | 0.7.0 | Kernel, daemon, CLI, all `crates/astrid-*` |
| `sdk-rust/` | `unicity-astrid/sdk-rust` | 0.7.0 | `astrid-sdk`, `astrid-sdk-macros`, `astrid-sys` |
| `astrid-rfcs/` | `unicity-astrid/rfcs` | n/a | Design proposals for the kernel-to-user-space contract |
| `capsules/astrid-capsule-agents/` | `unicity-astrid/capsule-agents` | - | Agent orchestration capsule |
| `capsules/astrid-capsule-cli/` | `unicity-astrid/capsule-cli` | 0.1.1 | Unix socket bridge for the CLI frontend |
| `capsules/astrid-capsule-fs/` | `unicity-astrid/capsule-fs` | - | Filesystem host-function capsule |
| `capsules/astrid-capsule-http/` | `unicity-astrid/capsule-http` | - | Outbound HTTP capsule |
| `capsules/astrid-capsule-identity/` | `unicity-astrid/capsule-identity` | - | Authentication and principal management |
| `capsules/astrid-capsule-memory/` | `unicity-astrid/capsule-memory` | - | Persistent agent memory |
| `capsules/astrid-capsule-openai/` | `unicity-astrid/capsule-openai` | - | OpenAI provider capsule |
| `capsules/astrid-capsule-openai-compat/` | `unicity-astrid/capsule-openai-compat` | - | OpenAI-compatible inference routing |
| `capsules/astrid-capsule-prompt-builder/` | `unicity-astrid/capsule-prompt-builder` | - | Prompt assembly |
| `capsules/astrid-capsule-react/` | `unicity-astrid/capsule-react` | - | Reactive event handling |
| `capsules/astrid-capsule-registry/` | `unicity-astrid/capsule-registry` | - | Model registry and active-model selection |
| `capsules/astrid-capsule-router/` | `unicity-astrid/capsule-router` | - | IPC topic routing |
| `capsules/astrid-capsule-session/` | `unicity-astrid/capsule-session` | - | Session lifecycle |
| `capsules/astrid-capsule-shell/` | `unicity-astrid/capsule-shell` | - | Shell command execution (`host_process` carve-out) |
| `capsules/astrid-capsule-skills/` | `unicity-astrid/capsule-skills` | - | Skill loader |
| `capsules/astrid-capsule-system/` | `unicity-astrid/capsule-system` | - | System-level capsule |
| `capsules/astrid-capsule-users/` | `unicity-astrid/capsule-users` | - | User management capsule |
| `capsules/astrid-capsule-context-engine/` | `unicity-astrid/capsule-context-engine` | - | Context window management |
| `capsules/astrid-capsule-hook-bridge/` | `unicity-astrid/capsule-hook-bridge` | - | Hook-bridge glue |

### Naming convention

Every capsule directory is named `astrid-capsule-<name>`. The GitHub repo drops the `astrid-capsule-`
prefix: `capsule-<name>`. The git remote inside each capsule directory reflects the shortened form.
When you need to push, the HTTPS URL is `https://github.com/unicity-astrid/capsule-<name>.git`.

You can verify any remote with `git remote get-url origin` from inside the capsule directory:

```bash
cd capsules/astrid-capsule-skills
git remote get-url origin
# https://github.com/unicity-astrid/capsule-skills.git
```

## Each Component Is Its Own Git Repo

This is the most important operational fact. `git` and `gh` commands resolve against the repo whose
working tree contains your current directory. You must `cd` into the target component before running
any git command. A `git status` at the polyrepo root shows the parent worktree's view, not the
kernel's or the SDK's.

```bash
# Wrong: operates on the parent worktree, not the kernel
git log --oneline -5

# Right: operates on the kernel repo
cd core
git log --oneline -5
```

The kernel workspace (`core/Cargo.toml`) lists 27 member crates from `crates/astrid-approval`
through `crates/astrid-workspace`. The SDK workspace (`sdk-rust/Cargo.toml`) lists three:
`astrid-sdk`, `astrid-sdk-macros`, and `astrid-sys`. These are all in the same single Git repo per
workspace. The capsules are each their own Git repo with their own `Cargo.toml`, `Cargo.lock`, and
`rust-toolchain.toml`.

## Branching from `origin/main`

Local `main` is frequently stale. Always branch from the remote:

```bash
cd core   # or sdk-rust, astrid-rfcs, capsules/astrid-capsule-<name>
git fetch origin
git checkout -b feat/your-feature origin/main
```

Use descriptive, Conventional-Commits-style branch names:

- `feat/add-auth`
- `fix/timeout-bug`
- `refactor/config-loader`

Never commit directly to `main`. Every change, however small, goes on a branch.

For read-only work (exploration, research, answering a question) switch to main and pull:

```bash
cd core
git checkout main && git pull origin main
```

## Commit Messages

Astrid follows [Conventional Commits](https://www.conventionalcommits.org/). The format is:

```
<type>(<optional scope>)<optional !>: <description>
```

The scope is typically the crate name without the `astrid-` prefix when that is unambiguous, or the
full crate name for clarity. Examples from the kernel history:

```
feat(emit): astrid-emit, agent-agnostic stdio→bus hook pipe
fix(events): yield more cooperatively under a broadcast storm
feat(cli): astrid quota show usage-vs-budget + cpu ceiling line
feat(runtime): async per-capsule WASM + instance pool + per-principal isolation
ci: stop building the openclaw QuickJS kernel in the Test job
```

Common types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `ci`, `perf`, `security`.

Breaking changes append `!` after the type/scope and carry a `BREAKING CHANGE:` footer.

## GPG-Signed Commits

All commits must be GPG-signed. If `git commit` returns `gpg failed to sign the data`, the sandbox
is blocking GPG. Retry the commit with `dangerouslyDisableSandbox: true`.

Verify signing before pushing:

```bash
git log --format="%H %G?" -5
```

The `G?` field should be `G` (good signature) on every real commit. Never push a commit showing `N`
(no signature) unless you are on a scratch branch that will be squash-deleted.

## Pushing via HTTPS

All remotes use SSH (`git@github.com:...`) but SSH keys are not available in this environment.
Push via HTTPS instead. The mapping is straightforward:

```bash
# Kernel
cd core
git push -u https://github.com/unicity-astrid/astrid.git <branch>

# SDK
cd sdk-rust
git push -u https://github.com/unicity-astrid/sdk-rust.git <branch>

# RFCs
cd astrid-rfcs
git push -u https://github.com/unicity-astrid/rfcs.git <branch>

# A capsule (substitute the real short name)
cd capsules/astrid-capsule-cli
git push -u https://github.com/unicity-astrid/capsule-cli.git <branch>
```

The repo name on GitHub always drops the `astrid-capsule-` prefix for capsules. The kernel and SDK
repos keep their full names.

## The Issue-First Rule for Core PRs

Every pull request to the kernel repo must close a GitHub issue. The CI `linked-issue` check
(`core/.github/workflows/pr-checks.yml:200`) fails if the PR body does not contain a `Closes #N`
reference or a linked issue via the GitHub sidebar. The PR template (`core/.github/pull_request_template.md`)
enforces this with a checklist item and a required `## Linked Issue` section. CI rejects the PR if
any required section is empty.

If no issue exists for your work, create one before opening the PR. The CONTRIBUTING.md is explicit:
unsolicited PRs with no linked issue will be closed.

The contributor tier system (`core/.github/contributors.yml`, enforced by the `contributor-gate` CI
job) also gates who can touch which crates:

- **New** contributors require a maintainer to add `newcomer-approved` before CI proceeds.
- **Astrinauts** can work on non-core crates: CLI, SDK, capsules, docs, and tests. They cannot
  modify `crates/astrid-crypto`, `crates/astrid-capabilities`, `crates/astrid-audit`,
  `crates/astrid-approval`, `crates/astrid-vfs`, `crates/astrid-storage`, `crates/astrid-sys`,
  or `crates/astrid-core`. They also cannot modify `crates/astrid-kernel`, `crates/astrid-events`,
  `crates/astrid-hooks`, `crates/astrid-config`, or `crates/astrid-mcp`. Both sets produce a hard
  CI error for Astrinauts.
- **Core** contributors can touch those crates but security-critical paths (`astrid-crypto`,
  `astrid-capabilities`, `astrid-audit`, `astrid-approval`, `astrid-vfs`, `astrid-storage`,
  `astrid-sys`, `astrid-core`) trigger a warning requiring maintainer co-review.
- **Maintainers** have full access.

The capsule repos and the RFC repo do not run the `contributor-gate` check. The issue-first rule
still applies in practice, but the CI enforcement lives only in the kernel repo.

## Filling In the PR Template

The `pr-checks.yml` workflow validates the PR body on every open, edit, reopen, and synchronize
event. All four sections must contain real content beyond the template placeholder comments:

```markdown
## Linked Issue
Closes #843

## Summary
Brief description of what the PR does and why.

## Changes
- Bullet list of notable changes.

## Test Plan
### Automated
- [ ] `cargo test --workspace` passes
- [ ] No new clippy warnings
```

A missing or blank section causes the `template` job to fail and blocks merge.

In addition, the `file-size` job rejects any source file that your PR pushes over 1000 lines if it
was under 1000 lines on `main`. A maintainer can override with the `large-file-ok` label.

## Version Bumps Are Separate PRs

A `chore: release X.Y.Z` commit is always its own pull request. Never include a version bump in a
feature or fix PR. This applies to the kernel, SDK, and every capsule equally.

## Pull Request Creation Is User-Initiated

Do not run `gh pr create` without being explicitly asked. Commit and push freely when work is
ready, but PR creation is the user's decision, not the agent's. PRs generate notifications and
signal intent to reviewers. Opening one without being asked makes that decision on someone else's
behalf.

If `gh pr create` fails with `you must first push the current branch to a remote`, supply the
`--head` flag:

```bash
gh pr create --head <branch-name> --title "..." --body "..."
```

## Building Across the Polyrepo

Build targets differ by component:

```bash
# Kernel (host target, Rust 1.95)
cd core && cargo build --workspace

# SDK (host target, Rust 1.94)
cd sdk-rust && cargo build --workspace

# One capsule (wasm32-unknown-unknown, Rust 1.94 per its rust-toolchain.toml)
cd capsules/astrid-capsule-cli && cargo build

# All capsules
for d in capsules/astrid-capsule-*; do (cd "$d" && cargo build); done
```

Capsules target `wasm32-unknown-unknown`. Their `rust-toolchain.toml` pins the toolchain and
declares the target. Do not pass `--target` manually; each capsule's `.cargo/config.toml` selects
the target for you. A capsule built under the kernel's `cargo` (a different toolchain, different
target) will not behave correctly.

The Rust edition across the kernel and SDK is 2024. The kernel MSRV is 1.95; the SDK MSRV is 1.94.

## The RFC Repo

`astrid-rfcs/` (`unicity-astrid/rfcs`) holds design proposals for the kernel-to-user-space contract.
You need an RFC for:

- Adding, removing, or changing a host function in `astrid-sys`
- Changing IPC topic conventions or payload schemas
- Modifying the capability token format or validation semantics
- Changing `Capsule.toml` manifest schema or dependency resolution
- Changing VFS path resolution rules or overlay behavior
- Defining a new capsule interface standard
- Breaking changes to `astrid-sdk` public API

You do not need an RFC for bug fixes, internal refactors that preserve external behavior,
documentation improvements, performance optimizations with no observable behavioral change,
adding a new capsule that implements an existing interface, or kernel-internal routing and storage
changes that keep the guest-visible WIT intact.

To submit an RFC:

```bash
cd astrid-rfcs
git fetch origin
git checkout -b rfc/my-feature origin/main
cp 0000-template.md text/0000-my-feature.md
# fill in the RFC
git add text/0000-my-feature.md
git commit -m "rfc: my-feature"
git push -u https://github.com/unicity-astrid/rfcs.git rfc/my-feature
```

Do not assign an RFC number. A maintainer assigns the next sequential number and renames the file
on merge. Discussion happens on the PR.

## Scope Discipline

Stay inside the named repo. If a task in `capsules/astrid-capsule-cli` requires touching a kernel
type, that is a signal to stop, flag the dependency, and not branch across repos to satisfy it. Any
change to the kernel-to-user-space contract surface requires an RFC first. Changes to host
functions, IPC payloads, capability tokens, or the manifest schema that you make inline in a capsule
PR without a corresponding RFC will be rejected.

## Checklist Before Pushing

1. `cd` into the right repo.
2. Branch from `origin/main`, not from a stale local `main`.
3. Branch name follows the `feat/`, `fix/`, `refactor/`, `docs/` convention.
4. Commits are GPG-signed. Verify with `git log --format="%G?" -1`.
5. `cargo test --workspace` passes (or `cargo build` for capsules, which lack a workspace test runner).
6. No new Clippy warnings.
7. `CHANGELOG.md` updated under `[Unreleased]`.
8. Core PR: a GitHub issue exists and the PR body contains `Closes #N`.
9. Version bump (if any) is on its own separate PR.
10. Push via HTTPS to the correct GitHub repo.

## See also

- [The Kernel-Is-Dumb Law](the-kernel-is-dumb-law.md)
- [The RFC Trigger](rfc-trigger.md)
- [Contribution Tiers and Security-Critical Crates](contribution-tiers.md)
