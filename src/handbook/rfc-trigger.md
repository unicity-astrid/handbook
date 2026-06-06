# The RFC Trigger

An RFC is required for any change to the contract surface between the kernel and user-space. It is not required for anything else. Getting this boundary right matters because RFCs gate ecosystem stability: every capsule author, every SDK consumer, and every third-party tool depends on the surfaces they describe. Changes inside those surfaces are implementation details and move freely. Changes to those surfaces affect everyone on both sides of the boundary and need a specification that lives outside any single crate.

The authoritative statement lives in `astrid-rfcs/README.md`:

> RFCs govern any substantial change to the contract surface between the kernel and user-space: the host ABI, IPC protocol, capability model, manifest schema, VFS semantics, capsule interface standards, and SDK public API.

RFC 0001 (`text/0001-rfc-process.md`) repeats and expands this into a scope table. Both documents are the ground truth. This page is a reading guide, grounded in the actual code, not a restatement of the documents.

## The seven in-scope areas

### 1. Host ABI

The host ABI is the set of WIT-typed functions the kernel exposes to capsules through the `astrid-sys` bindings. Every function in `wit/host/` is part of this surface.

The versioned WIT files are the canonical specification. Each carries an explicit freeze notice:

```
// Frozen per the ABI evolution discipline (RFC: host_abi). Shape changes
// ship as a new file at a new version path; never edit this file.
```

The current packages and a representative function from each:

| WIT package | File | Representative export |
|---|---|---|
| `astrid:sys@1.0.0` | `wit/host/sys@1.0.0.wit` | `random-bytes`, `log`, `get-caller` |
| `astrid:ipc@1.0.0` | `wit/host/ipc@1.0.0.wit` | `publish`, `subscribe`, `publish-as` |
| `astrid:fs@1.0.0` | `wit/host/fs@1.0.0.wit` | `fs-open`, `read-file`, `write-file` |
| `astrid:kv@1.0.0` | `wit/host/kv@1.0.0.wit` | `kv-get`, `kv-set`, `kv-cas` |
| `astrid:approval@1.0.0` | `wit/host/approval@1.0.0.wit` | `request-approval` |
| `astrid:identity@1.0.0` | `wit/host/identity@1.0.0.wit` | `identity-resolve`, `identity-link` |
| `astrid:net@1.0.0` | `wit/host/net@1.0.0.wit` | TCP, UDP, Unix socket access |
| `astrid:process@1.0.0` | `wit/host/process@1.0.0.wit` | `spawn` (desktop-kernel only) |
| `astrid:uplink@1.0.0` | `wit/host/uplink@1.0.0.wit` | `uplink-register`, `uplink-send` |
| `astrid:elicit@1.0.0` | `wit/host/elicit@1.0.0.wit` | Interactive install-time prompting |
| `astrid:http@1.0.0` | `wit/host/http@1.0.0.wit` | Outbound HTTP |
| `astrid:io@1.0.0` | `wit/host/io@1.0.0.wit` | `poll`, `input-stream`, `output-stream` |
| `astrid:guest@1.0.0` | `wit/host/guest@1.0.0.wit` | `astrid-hook-trigger`, `run`, `astrid-install` |

Adding a function, removing a function, changing a parameter type, changing an error variant, or changing the semantics of an existing call all require an RFC. The evolution discipline is: ship a new file at a new version path (`sys@1.1.0.wit`), never edit the frozen file.

The `astrid:guest@1.0.0` file is also in scope even though it defines exports (functions the kernel calls into the capsule). The shape of `astrid-hook-trigger`, `run`, `astrid-install`, and `astrid-upgrade` is just as contractual as the imports.

### 2. IPC protocol

The IPC bus carries JSON payloads over dot-delimited topics. Two things are in scope for the RFC process:

**Topic naming conventions.** The segment grammar (`[a-z0-9._-]+`, max 8 segments, max 256 bytes total) is defined in `wit/host/ipc@1.0.0.wit`. The existing topic namespaces (`user.v1.*`, `agent.v1.*`, `tool.v1.*`, `llm.v1.*`, `session.v1.*`, `registry.v1.*`, `client.v1.*`) are established contract. Adding a new top-level namespace or changing the `vN` versioning convention requires an RFC.

**Payload schemas.** The WIT interfaces under `wit/interfaces/` define the typed payload records for bus messages. Examples: `astrid-bus:types@1.0.0` (`types.wit`) defines `message`, `tool-call`, `tool-call-result`, and `tool-definition`. `astrid-bus:session@1.0.0` (`session.wit`) defines `get-messages-request`, `get-messages-response`, `clear-request`, and so on. `astrid-bus:tool@1.0.0` (`tool.wit`) defines `describe-request` and `describe-response`.

Capsules read and write these types. A schema change on either side breaks the other. Any payload field addition, removal, rename, or type change in `wit/interfaces/` requires an RFC.

### 3. Capability model

The capability model determines what a capsule is allowed to do. Two layers are in scope:

**Token format and validation semantics.** The kernel validates capability tokens on every host call. The `astrid:sys@1.0.0` `check-capsule-capability` host function (`wit/host/sys@1.0.0.wit`, lines 135-141) is the guest-visible API for capability introspection. Changes to how tokens are structured, how scopes are matched, or what the validation rules are require an RFC.

**Capability scope names.** The `[capabilities]` section of `Capsule.toml` uses a fixed vocabulary: `uplink`, `net_bind`, `ipc_publish`, `ipc_subscribe`, `host_process`, `fs_read`, `fs_write`, `identity`, and so on. The capability string a capsule declares must match what the kernel's ACL evaluator recognizes. Adding a new capability name, deprecating an existing one, or changing the scope semantics of an existing name requires an RFC.

### 4. Manifest schema

`Capsule.toml` is parsed into `CapsuleManifest` at `core/crates/astrid-capsule/src/manifest/mod.rs`. The struct definition is the manifest schema. The top-level sections are:

- `[package]` - identity metadata (name, version, description, authors, etc.)
- `[[component]]` - WASM entry points with id, file path, hash, and component-local capabilities
- `[capabilities]` - flat capability declarations (legacy form; `[publish]`/`[subscribe]` tables supersede)
- `[imports]` / `[exports]` - namespaced interface declarations (dual-form: flat `"ns:iface"` or nested `[imports.ns]`)
- `[publish]` / `[subscribe]` tables - cargo-like-manifest form for IPC ACL and interceptor bindings
- `[[tool]]` - tools surfaced to the LLM with `description_for_llm`
- `[[interceptor]]` - legacy interceptor bindings (superseded by `[subscribe]` handler field)
- `[[uplink]]`, `[[skill]]`, `[[mcp_server]]`, `[[command]]`, `[[context_file]]` - integration declarations
- `[env]` - environment variable elicitation with type, request, default, and enum_values

Adding a new top-level section, adding a new field that the kernel reads for routing or gating decisions, or changing how an existing field is interpreted at load time requires an RFC. Adding a field that is purely cosmetic (e.g. `[package].homepage`) is a lower bar, but the manifest is an untrusted input surface and any new field that crosses the operator/author trust boundary (see `EnvScope.scope`'s `skip_deserializing`) needs careful review and an RFC.

### 5. VFS semantics

The VFS presents capsules with scheme-rooted paths (`workspace://`, `home://`, `tmp://`). The semantics in scope include:

- Path resolution rules: how segments are validated, how symlinks are followed, what constitutes a boundary escape
- The scheme vocabulary itself: adding a new scheme like `cache://` requires an RFC
- The error contract: `error-code` variants in `astrid:fs@1.0.0` are part of the ABI

The `astrid:fs@1.0.0` host interface (`wit/host/fs@1.0.0.wit`) is frozen. Its freeze notice states that shape changes ship at a new version path. The existing `file-handle` resource, its `read-at` / `write-at` / `sync-data` / `sync-all` / `stat` / `set-len` methods, and the full set of path-based operations are contractual.

Notably: the freeze comment explicitly calls out that `fs-canonicalize` is not a security primitive (the kernel re-resolves every path on every call). This is a semantic guarantee. Changing that guarantee requires an RFC even if no WIT file changes.

### 6. Capsule interface standards

Interface standards govern how capsules talk to each other. The WIT records in `wit/interfaces/` define typed contracts that any capsule can implement or depend on. Examples: the `tool` interface (`astrid-bus:tool@1.0.0`) specifies the `describe-request`/`describe-response` handshake that every tool-providing capsule must honor. The `session` interface (`astrid-bus:session@1.0.0`) specifies the request/response protocol for conversation history.

Any capsule implementing these interfaces produces a binary that other capsules depend on. Adding a field, removing a field, or changing the topic naming convention for these inter-capsule protocols requires an RFC.

The `astrid:guest@1.0.0` `interceptor` world specifies the `action` string and `list<u8>` payload calling convention for `astrid-hook-trigger`. The `capsule-result` record (its `action` and `data` fields) is also a standard: every capsule that exports `astrid-hook-trigger` must return values the kernel knows how to interpret. Changing this contract requires an RFC.

### 7. SDK public API

The SDK (`sdk-rust/astrid-sdk`) is the primary dependency for capsule authors. Its module layout mirrors `std`: `fs`, `net`, `process`, `ipc`, `kv`, `http`, `elicit`, `identity`, `approval`, and `types`. The public re-exports, function signatures, and error types in each module are the SDK API.

Breaking changes include: removing a public function, changing a function signature, renaming a public type, changing the semantic contract of an existing function (e.g. changing when an error variant is returned), or reorganizing the module layout. All of these require an RFC.

Non-breaking additions (a new convenience method, a new optional parameter via a builder pattern) do not require an RFC, but they do require a semver minor bump and a CHANGELOG entry.

## What does not require an RFC

RFC 0001 is explicit (`text/0001-rfc-process.md`, lines 63-70):

> - Bug fixes to existing implementations
> - Internal refactoring that preserves the external contract
> - Documentation improvements
> - Performance optimizations that preserve existing behavior
> - Adding a new capsule that implements an existing interface
> - Kernel-internal changes that do not cross the ABI boundary

The key test is: does the guest-visible WIT surface change? If no capsule needs to recompile, and no `Capsule.toml` needs to change, and no IPC payload schema changes, the change is kernel-internal and needs no RFC.

Concrete examples of kernel-internal changes that do not require an RFC:

- Changing how the kernel dispatches IPC events internally (the bus routing algorithm, the broadcast fan-out implementation, the per-principal message queue data structure)
- Changing the storage backend for the KV store (e.g. switching from one embedded database to another) as long as the `astrid:kv@1.0.0` semantics are preserved
- Adding a new metrics endpoint to the gateway HTTP server
- Refactoring how the kernel validates capability tokens internally, as long as the external validation rules for capsules are unchanged
- Changing the WASM instance pool size, the blocking vs IO semaphore split, or the fuel ledger accounting algorithm
- Updating the daemon's shutdown sequence or signal handling

None of these touch the WIT files in `wit/host/` or `wit/interfaces/`, none change `CapsuleManifest`, and none change what capsule authors write in their `Capsule.toml` or their Rust code.

## The practical test

Before opening a PR, run through this checklist:

1. Does this change add, remove, or modify a function in any file under `wit/host/`? If yes, RFC required.
2. Does this change add, remove, or modify a type or record in any file under `wit/interfaces/`? If yes, RFC required.
3. Does this change add, remove, or modify a field in `CapsuleManifest` that the kernel reads for routing, gating, or capability decisions? If yes, RFC required.
4. Does this change alter the topic naming convention or payload schema for any IPC topic used across capsule boundaries? If yes, RFC required.
5. Does this change alter how capability scope names are matched or validated in a way visible to capsule authors? If yes, RFC required.
6. Does this change add or remove a VFS scheme, or change the path resolution or error semantics visible to capsule code? If yes, RFC required.
7. Does this change remove a public symbol from `astrid-sdk` or change a function signature? If yes, RFC required.

If the answer to all seven is no, the change is implementation-internal and can proceed without an RFC.

## Filing the RFC

The RFC repository is at `astrid-rfcs/`. The template is `astrid-rfcs/0000-template.md`. Process:

1. Fork the RFC repo and copy the template to `text/0000-my-feature.md`.
2. Fill in all required sections: Summary, Motivation, Guide-level explanation, Reference-level explanation, Drawbacks, Rationale and alternatives, Prior art, Unresolved questions, Future possibilities.
3. For any RFC that touches a host function or an IPC payload schema, the Reference-level explanation must include exact function signatures, input and output types with field types and constraints, error handling contracts, and ordering or concurrency guarantees. The spec must be precise enough for an independent developer to implement a conforming component from that section alone.
4. Open a pull request. The RFC number is assigned by a maintainer at merge time, not at PR time.
5. Once merged (status: Active), implementation proceeds in `astrid-sdk` behind a feature flag. Each RFC that defines types maps to a feature flag, e.g. `features = ["rfc-1"]`.

The lifecycle states are Draft (PR open), Active (merged, being implemented), Final (implemented and stable), Withdrawn (closed without merge), and Superseded (replaced by a newer RFC, noted in the header).

## See also

- [The Kernel-Is-Dumb Law](the-kernel-is-dumb-law.md)
- [Contribution Tiers and Security-Critical Crates](contribution-tiers.md)
