# The Kernel-Is-Dumb Law

The Astrid kernel has one job: route IPC events, enforce capabilities, and manage the WASM sandbox. It contains no business logic, no cognitive loops, no LLM glue, and no domain intelligence of any kind. That constraint is not an aspiration or a soft guideline. It is a hard architectural law that shapes every contribution to `core/`.

The module-level doc in `core/crates/astrid-kernel/src/lib.rs` states it plainly:

```rust
//! The Kernel is a pure, decentralized WASM runner. It contains no business
//! logic, no cognitive loops, and no network servers. Its sole responsibility
//! is to instantiate `astrid_events::EventBus`, load `.capsule` files into
//! the Extism sandbox, and route IPC bytes between them.
```

This page explains what the law means, why it exists, what it permits, what it forbids, and how to tell the difference in practice.

## The Rule

If the kernel does not route, gate, or validate it, it belongs in capsule-space IPC.

Three activities are the kernel's exclusive domain:

1. **Routing.** The kernel receives events on `EventBus` and delivers them to subscribers. The router in `kernel_router/mod.rs` listens on `astrid.v1.request.*` and dispatches `KernelRequest` variants. The admin router in `kernel_router/admin/mod.rs` does the same for `astrid.v1.admin.*`. Both are bus subscribers, not HTTP handlers or AI processors.

2. **Gating.** The kernel enforces capability checks before any request reaches a handler. Every `KernelRequest` variant maps to a capability string through `required_capability` (`kernel_router/mod.rs:488`). Every `AdminRequestKind` maps through `required_capability_for_admin_request` (`kernel_router/admin/mod.rs:190`). The preamble calls `authorize_request`, which reads `PrincipalProfile`, resolves group config from an `ArcSwap<GroupConfig>`, and runs `CapabilityCheck::require`. A missing or disabled principal is fail-closed: the check returns `PermissionError` and the request is rejected before any handler runs.

3. **Validating.** The kernel validates the wire types it owns: `PrincipalId` (alphanumeric, hyphen, underscore, 1-64 chars), `Quotas` (upper-bounded integers), capability grammar strings, profile TOML (strict `deny_unknown_fields`). None of this validation is semantic. The kernel does not know whether a quota value is right for a particular agent's use case. It only knows whether the value is well-formed.

Everything else is capsule business.

## What the Kernel Struct Looks Like

`Kernel` (`core/crates/astrid-kernel/src/lib.rs:44`) carries exactly the fields needed for its three jobs:

```rust
pub struct Kernel {
    pub session_id: SessionId,
    pub event_bus: Arc<EventBus>,
    pub capsules: Arc<RwLock<CapsuleRegistry>>,
    pub mcp: SecureMcpClient,
    pub capabilities: Arc<CapabilityStore>,
    pub vfs: Arc<dyn Vfs>,
    pub overlay_registry: Arc<OverlayVfsRegistry>,
    pub vfs_root_handle: DirHandle,
    pub workspace_root: PathBuf,
    pub home_root: Option<PathBuf>,
    pub cli_socket_listener: Option<Arc<Mutex<tokio::net::UnixListener>>>,
    singleton_lock: Option<std::fs::File>,  // private; held for daemon lifetime, Drop releases flock
    pub kv: Arc<SurrealKvStore>,
    pub audit_log: Arc<AuditLog>,
    active_connections: DashMap<PrincipalId, AtomicUsize>,
    fuel_ledger: FuelLedger,
    fuel_rate: FuelRateLimiter,
    pub ephemeral: AtomicBool,
    pub boot_time: Instant,
    pub shutdown_tx: watch::Sender<bool>,
    pub session_token: Arc<SessionToken>,
    token_path: PathBuf,
    pub allowance_store: Arc<AllowanceStore>,
    identity_store: Arc<dyn IdentityStore>,
    pub(crate) profile_cache: Arc<PrincipalProfileCache>,
    pub(crate) groups: Arc<ArcSwap<GroupConfig>>,
    pub(crate) astrid_home: AstridHome,
    pub(crate) admin_write_lock: Mutex<()>,
}
```

Every field serves routing, gating, or validation. `event_bus` is the backbone. `capsules` is the loaded WASM process table. `capabilities` and `profile_cache` are the gate. `audit_log` is the tamper-evident record of every gate decision. `fuel_ledger` and `fuel_rate` are per-principal CPU metering. `active_connections` counts clients for idle-shutdown logic. There is no `llm_client`, no `prompt_builder`, no `tool_registry`, no `conversation_history`.

## What Never Goes in the Kernel

**Domain-specific fields on `Kernel` or kernel types.** If you find yourself wanting to add a field that carries application state rather than routing or security state, stop. That field belongs in a capsule's KV namespace.

**Business logic in `KernelRequest` variants.** The current variants are install, approve-capability, list-capsules, reload-capsules, get-commands, get-metadata, shutdown, and get-status (`core/crates/astrid-core/src/kernel_api.rs:25`). They are lifecycle and introspection primitives. Adding a variant like `RunPrompt` or `BuildContext` would be a hard violation.

**Prompt assembly, model selection, tool dispatch, or session management.** These are capsule concerns. `capsule-session`, `capsule-prompt-builder`, `capsule-react`, and `capsule-skills` exist precisely so these concerns have a home outside the kernel.

**LLM provider integration.** The `astrid-capsule-anthropic` directory exists locally as a stale remnant of a capsule that was deleted. The kernel contains no provider logic, and capsule-side provider integration does not belong in `core/`.

**Inference about meaning.** The kernel routes bytes. It never looks inside an IPC payload to decide what it means for the application. The only payload inspection that happens is capability-checking: does the caller hold the required string? Even that check is mechanical, resolved entirely from `profile.toml` and `groups.toml`.

## The IPC Boundary

Capsules communicate exclusively through IPC events. Each capsule declares `[imports]` and `[exports]` in `Capsule.toml`. The kernel's `EventDispatcher` routes events to capsule interceptors. The kernel does not know and does not care what the interceptor does with the event.

The `AstridEvent::Ipc` variant carries every capsule-to-capsule message:

```rust
Ipc {
    metadata: EventMetadata,
    message: IpcMessage,
}
```

`IpcMessage` carries a topic string, an `IpcPayload` (typically `RawJson`), a sender `source_id` (UUID), and an optional principal. The kernel delivers it. The capsule decodes it, acts on it, and publishes a response. The kernel never decodes the JSON inside `RawJson`.

This design has a concrete consequence: if you want to add a new capability to the system, you add a capsule that subscribes to a new topic, not a new `KernelRequest` variant. Adding a topic is a capsule change. Adding a socket topic is also a capsule change: the Unix socket is owned by `capsule-cli`, which runs an uplink with a hardcoded allowlist, not by the kernel.

## The Capability-Enforcement Preamble

The one place the kernel inspects request meaning is the capability gate. The pattern is identical across both routers.

For `KernelRequest`:

```rust
let method = kernel_request_method(&req);
let scope = resolve_scope(&req, &caller);
let required_cap = required_capability(&req, scope);
match authorize_request(kernel, &caller, required_cap) {
    Ok(()) => { /* audit + continue */ },
    Err(e) => { /* audit + reject */ return; },
}
```

For `AdminRequestKind`:

```rust
let scope = resolve_admin_scope(&req.kind, &caller);
let required_cap = required_capability_for_admin_request(&req.kind, scope);
match authorize_request(kernel, &caller, required_cap) {
    Ok(()) => { /* audit + continue */ },
    Err(e) => { /* audit + reject */ return; },
}
```

`authorize_request` resolves the caller's `PrincipalProfile` from disk via `profile_cache`, checks the `enabled` flag (fail-closed on `false`), loads `GroupConfig` from the `ArcSwap` (a lock-free `Arc` clone), and runs `CapabilityCheck::require`. Every path produces an `AuditAction::AdminRequest` entry on both allow and deny, written to the chain-linked audit log and broadcast on `astrid.v1.audit.entry`.

The kernel's role in authorization ends here. It does not know why a principal needs `system:shutdown` or `self:capsule:install`. It only knows whether they have it.

## The `PrincipalProfile` Is Not a Session Object

`PrincipalProfile` (`core/crates/astrid-core/src/profile/mod.rs:112`) is static policy. It carries:

- `enabled`: master gate
- `groups`: group memberships resolved to capabilities via `GroupConfig`
- `grants` and `revokes`: per-principal overrides
- `auth`: accepted authentication methods and bound public keys
- `network`: egress allowlist
- `process`: spawn allowlist
- `quotas`: `Quotas` struct (memory, timeout, IPC throughput, background processes, storage, CPU fuel rate)

What it does not carry: conversation history, selected model, session preferences, active tool list, anything that varies per-request or per-session rather than per-principal-policy. Those belong in capsule KV or capsule-local state. The kernel reads `PrincipalProfile` at request time for the capability gate. It does not maintain a mutable runtime view of an agent's cognitive state.

Adding a new field to `PrincipalProfile` is only correct when the field represents a policy decision that the kernel actually enforces (routing, gating, or resource validation). A field the kernel reads but does not enforce is dead weight in a security-sensitive type. A field the kernel cannot interpret belongs in a capsule manifest or capsule KV.

## Worked Examples

**Correct: adding per-principal CPU budget enforcement.**

The fuel ledger (`fuel_ledger: FuelLedger`) and rate limiter (`fuel_rate: FuelRateLimiter`) fields on `Kernel` are legitimate kernel state. They gate WASM execution at the `invoke_interceptor` boundary. The kernel is the only actor that can enforce WASM execution limits because WASM runs inside the kernel's Wasmtime instance. The `max_cpu_fuel_per_sec` field on `Quotas` is a policy value the kernel reads to configure enforcement. This is routing and gating work.

**Incorrect: adding a preferred LLM model to `PrincipalProfile`.**

A `preferred_model: Option<String>` field on `PrincipalProfile` or `Quotas` would be domain configuration the kernel cannot interpret. The kernel cannot validate whether `"claude-sonnet-4-6"` is a real model. It cannot enforce that the capsule uses it. Model selection belongs in `capsule-session` or `capsule-react`, read from capsule KV or capsule manifest config, delivered to the LLM capsule via IPC.

**Correct: adding a `QuotaGet` admin request variant.**

`AdminRequestKind::QuotaGet { principal: PrincipalId }` is a read of the policy values the kernel owns. The handler reads `profile.toml` for the target principal and returns the `Quotas` struct. That is introspection of kernel-managed policy. The capability gate (`self:quota:get` or `quota:get`) and audit trail are already wired.

**Incorrect: adding a `RunTool` admin request variant.**

A variant that dispatches a tool call through the kernel would make the kernel an execution engine rather than a router. Tool dispatch belongs in capsule-space IPC: an agent publishes to a topic, a capsule intercepts, the capsule invokes the tool, the capsule publishes the result. The kernel never sees the tool arguments or result.

**Correct: the admin router dispatching group and agent management.**

`AgentCreate`, `AgentModify`, `CapsGrant`, `CapsRevoke`, `GroupCreate`, and similar variants write `profile.toml` and `groups.toml`. These files are the kernel's policy substrate. Managing them is the kernel's job. Every mutation goes through `admin_write_lock`, updates the `ArcSwap<GroupConfig>` atomically, and invalidates the `PrincipalProfileCache` entry. The kernel enforces these values at runtime; it must also own writing them.

**Incorrect: adding a field to `Kernel` for conversation context.**

A `context_store: Arc<ContextStore>` field on `Kernel` would make the kernel responsible for conversation state. That is `capsule-context-engine`'s job. The capsule publishes context read/write events on the bus. Other capsules subscribe. The kernel routes. Nothing in `Kernel` should know that prompts exist.

## The Capsule-Manifest Boundary

Capsule manifests (`Capsule.toml`) declare `[imports]` and `[exports]` that define the IPC surface a capsule participates in. The kernel reads manifests only for:

- Topological sort order at load time (`toposort_manifests`)
- Uplink detection (the `capabilities.uplink` flag determines load order)
- Import/export validation at boot (every required import must have a matching export)
- Interceptor subscription setup (`EventDispatcher` wires manifest interceptor patterns to bus subscriptions)

The kernel does not interpret manifest fields as application configuration. A manifest entry that says a capsule handles `prompt.v1.request.*` is meaningful to the dispatcher as a topic pattern. It is not meaningful to the kernel as a description of what "prompts" are.

Capsule manifests are untrusted input. Operator-only fields that affect cross-principal exposure require `skip_deserializing` and parser-isolation tests. The kernel's manifest parser rejects uplinks with `[imports]` sections as a defense-in-depth check (`lib.rs:576`), but the primary gate is `capabilities.uplink` validation at discovery time, not semantic understanding of what the imports mean.

## Uplinks Are Not Kernel Modules

Uplinks (CLI, web, Discord) are protocol clients that connect to the Unix socket. The socket is owned by `capsule-cli`, an uplink capsule with `net_bind` capability and a hardcoded topic allowlist. The kernel's role ends at the `UnixListener` (`cli_socket_listener: Option<Arc<Mutex<UnixListener>>>`), which is passed as context to the capsule at load time. The capsule performs the handshake, authenticates the session token, and bridges traffic onto the bus.

Adding socket handling logic to the kernel, or adding a new socket topic by changing a kernel constant, is a violation. Adding a socket topic means changing the capsule's allowlist. That is a capsule change.

## The Test Harness Confirms the Boundary

The `test_kernel_with_home` constructor (`lib.rs:925`) builds a `Kernel` with exactly the fields admin handlers touch: `event_bus`, `session_id`, `audit_log`, `profile_cache`, `identity_store`, `groups`, `astrid_home`, `admin_write_lock`, and the shared allowance, capability, and KV store handles. It skips socket binding, MCP init, token generation, and capsule discovery. The test harness compiles and passes without any application-level state. That is the surface area the kernel actually needs.

## Checklist for Kernel Contributors

Before adding any code to `core/crates/astrid-kernel` or `core/crates/astrid-core`:

1. Does the kernel route, gate, or validate this? If yes, proceed.
2. Does the kernel own the state this reads or writes? Policy substrate (profiles, groups, capabilities, audit log) is yes. Application state (conversations, model selections, tool results) is no.
3. Would removing this field break routing, capability enforcement, or the WASM sandbox? If no, the field does not belong in `Kernel`.
4. Does this add a `KernelRequest` variant that performs application work rather than lifecycle or introspection? If yes, it is a capsule IPC topic instead.
5. Is this a new socket topic? It belongs in `capsule-cli`'s allowlist, not in the kernel.
6. Does this change the kernel-to-user-space contract (host ABI, IPC protocol, capability model, manifest schema, VFS semantics, SDK public API)? If yes, write an RFC before touching code.

## See also

- [The RFC Trigger](rfc-trigger.md)
- [Working on Astrid: The Polyrepo and Git Workflow](polyrepo-and-workflow.md)
