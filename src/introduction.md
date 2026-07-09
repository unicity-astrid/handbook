# Introduction

If the agent lives inside the labyrinth, you are one of the people who build its walls. Build
them wrong and the whole promise leaks. This handbook is how the walls get built and kept.

This handbook is for people working on Unicity Astrid OS, not with it. If you are
building a capsule, the front-door documentation and The Unicity Astrid OS Book are
your references. If you are changing the kernel, the SDK, a contract surface, or a
reference capsule, read this first.

Unicity Astrid OS is a polyrepo. The kernel, the SDK, the RFCs, and each capsule are their own git
repositories under the `unicity-astrid` organization. Working on it means knowing which repo
owns what, what may never cross the kernel boundary, when a change needs an RFC, and how
releases are cut. This handbook is the operational law for that work.

## Three rules that govern everything

1. **The kernel is dumb.** The kernel routes events, enforces capabilities, and runs the
   sandbox. It holds no business logic. If the kernel does not route, gate, or validate a
   thing, that thing belongs in capsule-space IPC, not in a kernel type. This is not a style
   preference. It is the architecture, and violating it is how the system rots.
2. **RFCs are for contract changes.** An RFC is required when you change the surface between the
   kernel and user space: the host ABI, the IPC schema, the capability model, the manifest
   schema, VFS semantics, capsule interface standards, or the SDK public API. Kernel-internal
   changes that preserve the guest-visible contract do not need one. Know the difference before
   you open a PR.
3. **Ground in the code.** Documentation, including older READMEs in this very repository, drifts
   from the implementation. When the two disagree, the code wins, and the documentation is the
   bug. Verify against the source, not against prose.

## How releases and reviews work

Version bumps are always a separate pull request. Every core PR closes a GitHub issue. Commits
are GPG-signed. Security-critical crates carry extra review. After any non-trivial change, do an
adversarial self-review before requesting review: ask how this fails at three in the morning in
production, and what invariant it could violate. The chapters that follow make each of these
concrete.
