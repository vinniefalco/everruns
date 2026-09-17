<!-- source: everruns/everruns @ 6cf2c15e5 (crate everruns 0.20.0). The working tree fast-forwarded to 7d9e07a0b (0.21.1) during extraction; identifiers and examples were audited against 6cf2c15e5. -->
# Persistence and the Local Runtime

Every session you have built so far lasts only as long as the `Engine` that created it. That is the right default for tests and embedded tools, and it is the wrong default the moment a desktop app or a single-node service needs to pick up a conversation after its process was killed. The `local` feature closes that gap with one builder call: `.local(LocalConfig::new(data_dir))` gives an agent a crash-durable event log, a SQLite session catalog, task and schedule state, and a real-disk workspace, all under one directory you choose. Embedders who compose a host themselves get the same pieces as separate types, and the host contracts those pieces satisfy mark where a single-node deployment stops and the Everruns Platform begins.

## The engine-lifetime in-memory default and its limits

````rust
use everruns::prelude::*;

let agent = Agent::builder()
    .instructions("Remember what the user tells you.")
    .model(Model::simulated("noted"))
    .build()?;

let engine = Engine::new();
let session = engine.create(agent);
let session_id = session.session_id();
session.send_and_wait("My project is Atlas.").await?;
drop(session);

let reopened = engine.resume(session_id).await?;
reopened.send_and_wait("What is my project called?").await?;
````

With no storage mentioned anywhere on this agent, the engine runs entirely in memory. It owns a volatile session catalog and a volatile event log and needs no database, server, network connection, credential, or filesystem access. Dropping the `Session` value does not immediately discard its committed history. Because the engine retains the immutable agent snapshot associated with each session, passing the typed `SessionId` back to `engine.resume` on the engine that created it yields a live session with the earlier turn still in its history.

The limits of this default are two. A separate engine cannot infer a session's agent configuration; constructing a new `Engine::new()` starts a new volatile history store that knows nothing about sessions created elsewhere. Process exit loses everything.

You choose a persistence deployment by the recovery boundary you need, and there are only three.

- `Engine::new()` with no local config: volatile engine memory. Use it for tests and short-lived embedded tools.
- `Engine` with a `LocalConfig`: crash-durable local state. Use it for desktop apps and CLIs, or any other single-node service.
- The Everruns Platform: distributed durable execution. Use it for restarts, retries, horizontal workers, and remote clients.

The rest of this chapter is about the second option, crash-durable local state. The last section marks where the framework's host contracts stop and the Platform begins.

## Enabling the `local` feature and configuring `LocalConfig`

````text
cargo add everruns --features local
````

````rust
use everruns::prelude::*;

let agent = Agent::builder()
    .instructions("Remember what the user tells you.")
    .model(Model::simulated("durable reply"))
    .local(LocalConfig::new(".everruns-data"))
    .build()?;

let engine = Engine::new();
let session = engine.create(agent);
session.send_and_wait("My project is Atlas.").await?;
````

One feature and one builder call changed. The `local` feature pulls in the bundled `rusqlite` driver, which removes any need for a system SQLite install, and the platform crate. It exports `LocalConfig` at the crate root and in the prelude alongside `LocalGitWorkspaceProvider`, a Git-backed workspace provider that chapter 09 covers. Without the feature the `LocalConfig` type does not exist and `.local(...)` is not on the builder.

`.local(config)` attaches local state to the agent. From then on the engine reconstructs conversation history from the canonical local event log rather than from memory, and the first turn writes history under the configured data directory. The call also adds the `session_file_system` capability automatically; the agent gets a real-disk workspace without you naming that capability. Enable `local` and configure a trusted application data directory whenever sessions must survive a new agent or a new process.

`LocalConfig::new(data_dir)` is the only way to set the data directory; the field is not public and there is no setter. The workspace defaults to `workspace` inside that directory and is exposed to the agent as `/workspace`. To place the workspace somewhere else, chain `.workspace(root)` with a separate path.

````rust
use everruns::prelude::*;

let local = LocalConfig::new(".everruns-data").workspace("./workspace");
assert_eq!(local.data_dir(), std::path::Path::new(".everruns-data"));
assert_eq!(local.workspace_root(), std::path::Path::new("./workspace"));

let agent = Agent::builder()
    .instructions("Edit files in the project workspace.")
    .model(Model::simulated("done"))
    .local(local)
    .build()?;
````

Both paths read back as borrowed `&Path` values through `data_dir()` and `workspace_root()`, which is useful when the rest of the application needs to know where the agent's state is stored. The workspace root is independent of the data directory: a data directory under the user's application data folder can pair with a workspace inside a checked-out repository.

Under the data directory the runtime creates three things. `events.jsonl` is the crash-durable canonical event log. `local.db` is the SQLite catalog for session identity, task, and schedule state. `workspace` is the default real-disk workspace root.

Constraints on the data directory apply regardless of where you point it. Select both the data directory and the workspace root from trusted application configuration, never from model output or request input. The local profile is designed for one embedded process at a time; coordinate process ownership before handing the directory to another application process.

The application remains responsible for filesystem permissions, backups, retention, and the choice of directory. New local state files are created owner-only on Unix, but copied files and backups must still be protected by the application. History does not contain model credentials or application secrets unless the application deliberately puts them in message content or event metadata. Message content is application data and may be sensitive; choose and protect the directory accordingly.

Two examples in the `everruns` crate declare `required-features = ["local"]`. `session_history` persists, pages, resumes, and continues a session entirely offline with `Model::simulated` and a `LocalConfig`; no API key or network is involved. `workspace_heads` inspects workspace heads for local sessions and appears again in the resume section.

````text
cargo run -p everruns --features local --example session_history
cargo run -p everruns --features local --example workspace_heads -- /path/to/repo /path/to/state
````

## The SQLite database and the local session catalog

````rust
use everruns::local::{LocalSessionStore, SqliteDb};

let db = SqliteDb::open(data_dir.join("local.db"))?;
let catalog = LocalSessionStore::new(db.clone())?;

let db = SqliteDb::open_in_memory()?;
let throwaway = LocalSessionStore::new(db)?;
````

Under `LocalConfig` the engine makes these two calls for you, and together they are the whole database layer. `SqliteDb::open(path)` opens one file-backed SQLite database, creating it if needed, and that handle is the shared storage for every local store. `SqliteDb::open_in_memory()` gives an in-memory database for tests and throwaway runs. `LocalSessionStore::new(db)` creates the catalog schema on first open and reuses it on later opens of the same file; calling the constructor on every start is safe.

`SqliteDb` derives `Clone`. A clone shares the underlying connection, which means every store built from the same handle writes to the same file. That connection is single and serialized behind a mutex, which suits the embedded, single-process hosts with modest throughput that the local stores target.

The database file itself comes with durability and privacy guarantees that hold on every open. Connections open in WAL journal mode, which lets a freshly spawned process reopen the same file after a restart without losing committed data. Foreign keys are enforced, and short lock contention waits up to 5 seconds before failing. On Unix the file is created with owner-only permissions (mode `0600`). The runtime rejects a symlinked database path, rejects a path that is not a regular file such as a directory or device, rejects a file owned by a different Unix user, and tightens group- or world-accessible permissions on an existing file back to owner-only. On non-Unix platforms these checks are skipped.

The session catalog is a durable record of session identity for local hosts, and what it stores is deliberately minimal. The `framework_sessions` table has a single column, `session_id`. Full runtime session configuration stays in memory and is rebuilt from the resuming agent; MCP credentials and other host configuration are never serialized. A framework test runs a session whose MCP server uses a bearer-token header, then scans every file under the data directory and asserts the secret never reached disk. The catalog tables hold no message or event columns either.

Reopening a persisted id after a restart yields a fresh runtime session shell for that id, not a placeholder and not an error, and its title is not restored. Looking up an id the catalog does not know returns `None` rather than a synthesized session; unknown sessions stay distinguishable from known ones. Registering a new session inserts its id with `ON CONFLICT DO NOTHING`, and re-adding an existing id is therefore a no-op at the SQLite layer.

### Workspace bindings and runtime mutations

A workspace binding is the opaque, credential-free record of which provider, workspace, and head a session was using; workspaces and heads themselves are chapter 09's subject. The catalog stores the binding to let the exact workspace head be reopened after restart. `bind(session_id, &binding)` records a binding for a session and `load(session_id)` returns `Ok(None)` when none exists. The binding survives reopening the file, and binding a workspace registers the session id without a separate add call.

A head claimed as `WorkspaceHeadAccess::Isolated` is owned by exactly one session; a second session binding the same isolated head receives `EnvironmentBindingError::Conflict`. A head claimed as `WorkspaceHeadAccess::Shared` accepts multiple sessions, and a head cannot switch between isolated and shared. Rebinding the same session with the identical binding is idempotent; a different binding for that session is a conflict. Binding payloads have a hard size ceiling, and oversized or undecodable bindings are rejected as `EnvironmentBindingError::Corrupt`. Storage errors surface as `EnvironmentBindingError::Unavailable`, which keeps infrastructure failure distinguishable from conflict and corruption.

At runtime a session's title can be set with `update_session_title(session_id, title)`, a capability can be added or replaced by id with `upsert_session_capability(session_id, capability)`, and one can be removed with `remove_session_capability(session_id, capability_id)`. These mutations are held in memory only and are not restored after a restart. Mutating an id the catalog does not know produces a store error reading `session not found: {session_id}`.

## The crash-durable event log and resuming after restart

````rust
use everruns::prelude::*;

fn build_agent() -> Result<Agent, BuildError> {
    Agent::builder()
        .instructions("Remember what the user tells you.")
        .model(Model::simulated("durable reply"))
        .local(LocalConfig::new(".everruns-data"))
        .build()
}

// Process one: create, persist, exit.
let first_engine = Engine::new();
let session = first_engine.create(build_agent()?);
session.start().await?;
let session_id = session.session_id();
session.send_and_wait("My project is Atlas.").await?;
session.send_and_wait("Remember that name.").await?;
// Store `session_id` in trusted application state, then the process exits.

// Process two: rebuild, attach, resume, continue.
let engine = Engine::new();
engine.attach(session_id, build_agent()?).await?;
let resumed = engine.resume(session_id).await?;

let page = resumed.history().page().await?;
assert_eq!(page.len(), 4);
resumed.send_and_wait("Continue the session.").await?;
````

This is the shape of the `session_history` example. Process one creates a session and runs two turns, which the local event log records as four messages. Process two knows nothing except the `SessionId` and how to build the agent; because the catalog stores no agent configuration, trusted application configuration is the only place a rebuilt agent can come from. Process two constructs a fresh `Engine::new()`, attaches the rebuilt agent to the persisted id with `engine.attach(session_id, agent)`, resumes by that id with `engine.resume(session_id)`, and continues; the third turn appends to the same durable transcript. A framework test runs three separate engines against one directory in this way, with the first creating an empty session, the second adding a turn, and the third seeing two messages and adding a turn to reach four.

### When durability begins

Constructing an engine opens nothing. Creating a session with `engine.create(agent)` opens nothing either; backend initialization is deferred until a session needs it. A new session becomes durable at its first async operation. A run counts, as does an inspect or a history page read. Merely holding a session handle does not commit it. To materialize a session before sending any input, call `session.start().await?`, as the example does before reading `session_id()`. At that point the runtime ensures the directories exist and opens both the event log and the SQLite catalog.

Agent events are persisted and replayed through a pluggable event log with two shipped implementations: `InMemoryEventLog` for tests and `JsonlEventLog`, an append-only JSON Lines file. Under `LocalConfig` the engine opens `JsonlEventLog` for you. Conversation history is a read-only projection rebuilt from replaying that log, and normal execution has one write path, the engine's event log. That is why a resumed session and a running session can never diverge on the conversation. The file format and the host backends behind it are not framework APIs; do not edit the log or build application writes around its representation.

### Resume outcomes

````rust
use everruns::prelude::*;

match engine.attach(session_id, build_agent()?).await {
    Ok(()) => {}
    Err(ResumeError::SessionNotFound { session_id }) => {
        eprintln!("no local record for {session_id}");
    }
    Err(ResumeError::Unavailable) => {
        eprintln!("local data directory or catalog cannot be used");
    }
    Err(ResumeError::Corrupt) => {
        eprintln!("events.jsonl contains a complete non-JSON record");
    }
    Err(other) => return Err(other.into()),
}
````

`attach` looks the id up in the session store. It rejects ids absent from that agent's configured local catalog with `ResumeError::SessionNotFound { session_id }`, which includes the id. When the store cannot be queried, or the data directory cannot be used because the configured path is a regular file, the error is `ResumeError::Unavailable`; infrastructure failures are kept separate from missing and corrupt state. Tests can compare `ResumeError` values directly because the type implements `PartialEq`.

- Attach is idempotent. Attaching an id the engine already knows is a no-op success, and the first attached agent wins. A second attach with a different agent snapshot succeeds but does not replace the first.
- A torn trailing record in `events.jsonl`, a partially written last line after a crash, is recovered automatically on resume with every committed turn preserved. A complete but non-JSON line is genuine corruption and resume fails with `ResumeError::Corrupt`.
- Crash recovery is bounded by `MAX_JSONL_RECOVERY_BYTES` and `MAX_JSONL_RECOVERY_EVENTS`: at most 128 MiB and 1,000,000 canonical events. Oversize logs fail to open with a typed recovery-limit error instead of scanning without bound.
- Every live engine configured with the same local data directory shares one backend bundle within a process, so concurrent engines cannot build divergent event-log indexes or SQLite handles. Relative paths, paths containing `..`, and platform-aliased paths such as macOS `/var` and `/private/var` resolve to the same shared backend. A turn written by a second engine is visible in the first engine's history.
- Two agents opening the same local profile with different workspace roots produce a configuration error reading `local profile {} is already open with workspace {}; requested {}`, surfaced to the caller as `ResumeError::Unavailable`.

The `workspace_heads` example resumes local sessions with a workspace attached. It takes a repository path and a state root as arguments; the state root defaults to `.everruns-local` inside the repository and holds `runtime` and `workspace-provider` subdirectories, with the agent configured as `.local(LocalConfig::new(state.join("runtime")))`.

## `LocalProfile`, `LocalBackends`, and the local runtime builder

````rust
use everruns::local::{LocalProfile, LocalRuntimeBuilder};

let profile = LocalProfile::new("/var/lib/myapp/everruns");
let (runtime, local) = LocalRuntimeBuilder::new(profile).build().await?;
````

This section is for embedders who compose a host themselves instead of going through `Agent::builder().local(...)`. The in-process runtime that `build()` returns has its own chapter, 10. This section is about the three local types that stand it up over SQLite, all of them strictly optional: everything the builder does can be done by composing the pieces directly, and `LocalConfig` uses the same pieces under the engine.

A `LocalProfile` is a single value bundling the on-disk locations and identity defaults an embedder needs: `data_dir`, `workspace_root`, `org_public_id`, and `owner_principal_id`. All four fields are public; the data directory is set through `new`, and the other three have chainable setters.

````rust
use everruns::local::LocalProfile;

let profile = LocalProfile::default();
println!("{}", profile.db_path().display());

let profile = LocalProfile::new("/var/lib/myapp/everruns")
    .with_workspace_root("/srv/checkouts/project")
    .with_org_public_id("org_myapp")
    .with_owner_principal_id(owner);
profile.ensure_dirs()?;
````

The zero-configuration profile is `LocalProfile::default()`. It roots state in the user's platform-native local data directory plus `everruns/local`: on Linux `$XDG_DATA_HOME` when set to an absolute path, otherwise `~/.local/share`; on macOS `~/Library/Application Support`; on Windows `%LOCALAPPDATA%`. When no home or data directory can be resolved at all, it falls back to the system temp directory. The workspace goes to `workspace` under the data directory, `org_public_id` is the framework's default public organization id, and `owner_principal_id` is the fixed local principal `PrincipalId::from_seed(1)`. The default location is deliberately kept byte-identical to what the `dirs` crate returns, because moving it would orphan existing state.

Pass your own directory to `LocalProfile::new(data_dir)` and the workspace moves to `workspace` under it while every other field keeps its default. `db_path()` answers where the database file is. `ensure_dirs()` creates the data and workspace directories with private permissions in one call. On Unix it rejects a symlinked directory, rejects a directory owned by another user, rejects a path that is not a directory, and tightens permissions to owner-only (mode `0700`); on Windows it simply creates the directories. Failures surface as configuration errors rather than panics.

````rust
use everruns::local::{LocalBackends, LocalProfile, SqliteDb};
use everruns_host::HostBackends;

let local = LocalBackends::new(profile.clone(), HostBackends::in_memory())?;

let db = SqliteDb::open_in_memory()?;
let for_tests = LocalBackends::with_db(profile, HostBackends::in_memory(), db)?;
````

Local backends come from a profile in one call. `LocalBackends::new(profile, runtime_backends)` creates the directories if missing, opens the database at `profile.db_path()`, and returns a complete set of SQLite-backed runtime backends. `HostBackends` is the host crate's bundle of non-filesystem stores, covered in the last section of this chapter and in chapter 10; you pass in the bundle you already have, and only the local task registry and schedule store factory are attached on top of it. You keep your own event bus and your own session filesystem factory. `with_db` accepts an already-open handle such as an in-memory database for tests. The composed `runtime_backends`, the shared `db` handle, the `task_registry`, and the `profile` are all public fields, which leaves you free to wire only the pieces you want, and `org_id()` returns the internal organization id derived deterministically from the profile's public one.

A crash-durable local runtime comes out of `LocalRuntimeBuilder::new(profile)` without your hand-composing local backends and a host composition on top of the runtime builder; it wraps the in-process runtime builder. Each method configures one slot before `build()`.

- `.host_composition(HostComposition)` replaces the entire platform definition (capabilities, drivers, egress, filesystem factory); capability and driver overrides then take effect instead of being silently discarded. When you supply one, you own the session filesystem factory on it.
- `.provider_with_default_model(provider, model_id)` registers an LLM provider and makes one of its models the runtime default in one call, replacing any same-name provider registration. Deterministic local runs pass the simulator provider here.
- `.default_model(ModelSpec)` sets the runtime-wide default model explicitly.
- `.harness(SeededHarness)`, `.agent(AgentDefinition)`, and `.session(ExecutionSession)` seed a harness under an id you choose, agent definitions, and execution sessions at build time.
- `.session_file_system_factory(factory)` swaps the default real-disk session filesystem for a custom implementation, such as an in-memory or sandboxed one.
- `.inner_mut()` exposes the underlying `InProcessRuntimeBuilder` for wiring the local builder does not expose directly.
- `.build().await` returns `(InProcessRuntime, LocalBackends)`; the second value lets the same SQLite-backed task registry and schedule store be reused elsewhere in the application.

`local_capability_registry()` returns the exact capability registry a local profile executes with. A hand-built host composition that starts from it passes the same build-time capability validation as the local runtime, because the local profile serves the hosted catalog from SQLite and validation and execution use one set. The default local registry includes, among others, `session_schedule`, `current_time`, `compaction`, `tool_call_repair`, and `usage_limit_auto_continue`.

## Local error variants

````rust
use everruns::local::{LocalError, LocalResult, SqliteDb};
use std::io::ErrorKind;
use std::path::Path;

fn open_catalog_db(path: &Path) -> LocalResult<SqliteDb> {
    SqliteDb::open(path)
}

match open_catalog_db(&db_path) {
    Ok(db) => run_with(db).await?,
    Err(LocalError::Io(err)) if err.kind() == ErrorKind::PermissionDenied => {
        eprintln!("cannot open local state: fix permissions on {}", db_path.display());
    }
    Err(LocalError::Sqlite(err)) => {
        eprintln!("sqlite refused the database: {err}");
    }
    Err(LocalError::Config(msg)) => {
        eprintln!("bad local configuration: {msg}");
    }
    Err(other) => return Err(other.into()),
}
````

Every failure the local SQLite-backed runtime can produce is one enum, `LocalError`, and `LocalResult<T>` is the alias pairing it with any success type. The enum derives `Debug` and implements the standard `Error` and `Display` traits; it can be boxed like any other error or matched as above. All public `SqliteDb` functions return `LocalResult`, and the `#[from]` conversions mean `?` on a `rusqlite::Result` or `std::io::Result` inside local code converts automatically. Each variant has a fixed display prefix; log lines identify the failure class without a type match.

- `Sqlite`: a storage error, displayed as `sqlite error: ...`, wrapping the original `rusqlite::Error` for cases such as a busy or locked database or a constraint violation.
- `Serde`: a persisted JSON value could not be encoded or decoded, displayed as `serialization error: ...`.
- `Io`: a filesystem error, displayed as `io error: ...`. The payload is a `std::io::Error` whose `.kind()` distinguishes permission denied from other failures, as the example does.
- `Config`: the requested local configuration is invalid, displayed as `configuration error: ...`.
- `Other`: a catch-all that displays its raw message with no prefix, so custom local components can report errors without defining new types.

At the engine boundary `LocalError` converts into `AgentLoopError`, the agent-loop error enum from chapter 04. `LocalError::Config(msg)` becomes an agent-loop configuration error, and every other variant becomes a store error. The typed inner error is flattened to a string at that boundary: after conversion there is no `rusqlite::Error`, `serde_json::Error`, or `std::io::Error` left to inspect. Code that needs the original driver, JSON, or I/O error must match `LocalError` before that conversion, which is why the example matches at the `SqliteDb::open` call rather than on the engine's result.

## Delegating to other agents through the local platform store and the `a2a` opt-in

````rust
use std::sync::Arc;
use async_trait::async_trait;
use everruns::local::{LocalPlatformStore, LocalSessionRunner};

struct RuntimeRunner { /* wraps the InProcessRuntime built earlier */ }

#[async_trait]
impl LocalSessionRunner for RuntimeRunner {
    async fn create_session(
        &self,
        harness_id: HarnessId,
        agent_id: Option<AgentId>,
        title: Option<&str>,
        locale: Option<&str>,
        parent_session_id: Option<SessionId>,
    ) -> Result<ExecutionSession> { /* create a real local session */ }

    async fn send_message(&self, session_id: SessionId, content: &str) -> Result<()> {
        /* deliver the message and run the turn to completion */
    }

    async fn list_sessions(&self, limit: Option<usize>, agent_id: Option<AgentId>) -> Result<Vec<ExecutionSession>> { /* ... */ }
    async fn get_session(&self, session_id: SessionId) -> Result<Option<ExecutionSession>> { /* ... */ }
    async fn get_messages(&self, session_id: SessionId, limit: Option<usize>) -> Result<Vec<PlatformMessage>> { /* ... */ }
    async fn get_session_status(&self, session_id: SessionId) -> Result<Option<String>> { /* ... */ }
}

let store = LocalPlatformStore::new(Arc::new(RuntimeRunner { /* ... */ }));
````

Chapter 07 introduced `spawn_agent` as a hosted Platform capability. Subagent tools call through a platform-store seam, and in a fully local runtime you enable subagent spawning by adapting your own session runtime into that seam. `LocalPlatformStore` implements the subagent-critical core of the platform store contract, backed by a `LocalSessionRunner` you supply. The embedder wires the runner to its in-process runtime; through the runner the store creates real local sessions and drives them by sending messages and waiting for idle. The store itself never owns the runtime.

The six required methods are the ones in the example. `routable_session_ids()` is an optional override defaulting to `Ok(None)`, which means every session in the organization is routable. Embedded hosts with a narrower or dynamic route set return `Some`, including an empty vector when no session is active, and schedule claims stay scoped to work the host can deliver. `LocalPlatformStore::new(runner)` is the only constructor; it takes no base URL. The store derives `Clone`; a clone is a reference-count bump on the runner, cheap enough to share one store freely.

The local store implements the contract with limits that a caller observes directly. `create_session` takes a harness, an optional agent, a title, a locale, and an optional parent session link, and returns the portable execution view. Only a fresh seed is supported locally: requests that fork from a session, set a budget root, or reference a blueprint are rejected explicitly with an error naming `create_session(seed)` or `create_session(blueprint)`, never silently degraded. `send_message` delivers a user message and runs the turn to completion before returning, and because of that `wait_for_idle` returns immediately, ignores its timeout argument, and reports session not found for an unknown session.

Session records read back through the store are attributed to the fixed local principal, since the local host has no real session ownership catalog. Platform-management calls the local store does not support fail with `operation '{op}' is not supported by the local platform store; manage this entity in embedder code`. Harness lookup, agent lookup, and adding an agent participant are the operations that return this error.

Wiring is a two-step process. `LocalRuntimeBuilder` leaves the platform store unwired because the store needs a runner that usually wraps the built runtime. Build the runtime first, then attach the runner to the local backends with `LocalBackends::with_platform_runner(runner)` and rebuild, or use the standalone `LocalPlatformStore` directly.

````text
cargo add everruns --features a2a
````

The `a2a` feature enables outbound delegation to remote agents over the Agent2Agent protocol. It is off by default; hosts that delegate to remote agents opt in explicitly. It is defined as `a2a = ["local", "everruns-platform/a2a"]`, which means it requires and includes `local`, because the delegation capability ships in the platform crate. Two identifiers are available as constants in the core crate's capabilities module even when the feature is compiled out: `A2A_AGENT_DELEGATION_CAPABILITY_ID`, whose value is `a2a_agent_delegation`, and `AGENT_RUN_KEY_PREFIX`, whose value is `agent_run:`. Session logic can reference them without a feature gate.

## Execution loading contracts and resolved snapshots

````rust
use std::sync::Arc;
use async_trait::async_trait;
use everruns_core::execution_loading::{AgentStore, HarnessStore, SessionStore};
use everruns_core::{AgentDefinition, ExecutionSession, HarnessDefinition, ResolvedExecutionSnapshot};

struct FileStores { /* your persistence */ }

#[async_trait]
impl AgentStore for FileStores {
    async fn get_agent(&self, agent_id: AgentId) -> Result<Option<AgentDefinition>> {
        /* Ok(None) when missing; Err when archived or deleted */
    }
}

#[async_trait]
impl HarnessStore for FileStores {
    async fn get_harness(&self, harness_id: HarnessId) -> Result<Option<HarnessDefinition>> { /* ... */ }
}

#[async_trait]
impl SessionStore for FileStores {
    async fn get_session(&self, session_id: SessionId) -> Result<Option<ExecutionSession>> { /* ... */ }
}

let stores: Arc<FileStores> = Arc::new(FileStores { /* ... */ });
let harness = stores.get_harness(session.harness_id).await?.expect("harness exists");
let agent = match session.agent_id {
    Some(id) => stores.get_agent(id).await?,
    None => None,
};
let snapshot = ResolvedExecutionSnapshot::project(&harness, agent.as_ref(), &session)?;
````

Three small storage contracts let you plug in your own persistence for turn execution, and each has a single required method. `AgentStore::get_agent` returns a portable `AgentDefinition` for a public `AgentId`. `HarnessStore::get_harness` returns a `HarnessDefinition` for a `HarnessId`, or `Ok(None)` when the harness does not exist. `SessionStore::get_session` returns a portable `ExecutionSession` for a `SessionId`. All three require `Send + Sync`, use `#[async_trait]`, and have blanket implementations for `Arc<T>` including unsized `T`. That is what allows an `Arc<dyn AgentStore>` to be shared across tasks on a multi-threaded executor. `LocalSessionStore` from earlier in this chapter implements `SessionStore`, which is how the execution loader loads local sessions the same way as any other store. Beyond the method signatures, a store implementer is bound by a small set of contracts.

- Not found and cannot execute are distinct outcomes. A missing record returns `Ok(None)`. A record that exists but is archived or deleted must return an error; lifecycle validation then happens at the loading seam before host execution begins.
- `get_agent_blocker` and `get_harness_blocker` report why an agent or harness cannot run, returning `None` when it is executable or a `DependencyBlocker` (`AgentDeleted`, `AgentArchived`, `HarnessDeleted`, `HarnessArchived`) describing the problem. The defaults treat a missing record as deleted, and a custom store gets blocker detection with no extra code.
- Parent-chain harness inheritance is resolved behind `get_harness`, root to leaf, using the same overlay merge the runtime uses. Callers receive one effective harness and never walk the chain themselves.
- The `ExecutionSession` contains only what a turn needs: correlation values and the per-session configuration layer, together with neutral execution state. The full persisted session aggregate with its facets, participants, ownership summaries, timestamps, and UI metadata stays behind the store. The core loading contracts are read-only; mutating stored session metadata is a hosted control-plane concern.

### The resolved execution snapshot

`ResolvedExecutionSnapshot::project(&harness, agent, &session)` projects a harness definition, an optional agent definition, and an execution session into one canonical value that host turn execution consumes. The framework in-process runtime and hosted workers build the same value, which puts precedence in one code path, `AgentConfigOverlay::fold`, applied in the order harness, then agent, then session, leaf wins, exactly once. The per-field merge rules from chapter 05 land here.

- Instructions concatenate root to leaf. A session model overrides an agent default model, which overrides the harness model, and no model at any layer selects the host default.
- Capabilities override by id while capabilities added by other layers are kept. A capability persists as `{"ref": "web_fetch", "config": {"enable_file_download": true}}` and round-trips through projection unchanged.
- Initial workspace files override by path. Scoped MCP servers override by name. Network access narrows across layers: the allow list intersects and the block list accumulates.
- `max_iterations` and `parallel_tool_calls` take the leaf-most value, including an explicit zero or `false`.

The snapshot includes typed ids for `session_id`, `workspace_id`, `harness_id`, optional `agent_id`, and `organization_id`, plus `locale`, `tags`, `blueprint_id` and `blueprint_config`, `cumulative_usage` from prior turns, `embedder_metadata` folded from the harness chain, and the explicitly configured `tools`. MCP servers in scope appear as `SnapshotMcpServer` values with transport, `header_names`, `env_names`, auth mode, protocol mode, `oauth_provider_id`, and `tool_discovery` flag. Header and environment values never enter the snapshot.

A session that references an agent which was not supplied fails with `AgentLoopError::agent_not_found(agent_id)` before execution begins. A mismatched agent is a configuration error reading `session {} references agent {} but agent {} was provided`. An unreferenced agent contributes nothing; projecting with it serializes identically to projecting without it.

Serializing a snapshot with `serde_json` or printing it with `Debug` is guaranteed to contain no credential values (MCP URL userinfo, header values, environment values, command, arguments) and no UI or platform metadata such as the session title or agent description. `Debug` output additionally redacts capability config payloads. Field order is fixed and map fields are `BTreeMap`s, so equal snapshots serialize to byte-identical JSON regardless of input insertion order; snapshots can be persisted, hashed, diffed, and replayed. The host crate exports `load_execution_snapshot` and `load_execution_snapshot_for_session` to read snapshots back from disk, whole or per session.

## Host backends, the example hosts, and the distributed seam

````rust
use std::sync::Arc;
use everruns_host::{HostBackends, InProcessRuntimeBuilder};
use everruns_host::events::JsonlEventLog;

let log = Arc::new(JsonlEventLog::open(data_dir.join("events.jsonl")).await?);

let backends = HostBackends::in_memory()
    .with_event_log(log.clone())
    .with_event_sink(my_sink)
    .with_session_store(my_session_store);

let runtime = InProcessRuntimeBuilder::new()
    .backends(backends)
    .build()
    .await?;
````

`HostBackends` is the bundle that supplies every non-filesystem store to an execution host, and it is defined in the host crate that chapter 10 covers in depth. `HostBackends::in_memory()` produces a complete set in one call: `InMemoryHarnessStore`, `InMemoryAgentStore`, `InMemorySessionStore`, `InMemoryEventLog`, `InMemoryCompactionCheckpointStore`, `InMemoryProviderStore`, `InMemorySessionStorageStore`, and a `NoopEventSink`, with every optional slot `None`. Tests and examples run on that set, as does the default runtime. The bundle exists so you can keep the public runtime orchestration but bring your own store implementations, either by handing a whole bundle to `InProcessRuntimeBuilder::backends(...)` or by overriding individual stores with the chainable `with_*` setters starting from the in-memory defaults. The struct derives `Clone`. Each slot below is a public field on the bundle with a `with_*` setter of the same name.

- `event_log`: `InMemoryEventLog`, `JsonlEventLog`, or your own `EventLog` implementation.
- `event_sink`: a non-blocking live observation sink notified after each event commits, for streaming or monitoring. The default does nothing.
- `session_store`, plus `with_agent_store`, `with_harness_store`, and `with_provider_store`: runtime stores that extend the read-only loading contracts with a seeding method. `RuntimeAgentStore` adds `add_agent`, `RuntimeHarnessStore` adds `add_harness`, `RuntimeSessionStore` adds `add_session` and includes `SessionMutator`, and `RuntimeProviderStore` adds `set_default_model_spec`. The built-in in-memory stores satisfy these without adapter code.
- `storage_store`: per-session key-value and secret storage through `SessionStorageStore`.
- `compaction_checkpoint_store`: durable replacement context used to reconstruct compacted model input.
- `native_async_store`: a shared fenced journal that enables native asynchronous tool execution. Unset by default.
- `connection_resolver`: a resolver for user connection tokens such as GitHub or Daytona, fetched lazily at tool time. There is no default because a resolver implies a real credential source.
- `session_task_registry`: persists the lifecycle of background tools, subagents, and monitors. Leaving it unset keeps prior behavior.
- `with_schedule_store_factory`: a `ScheduleStoreFactory` closure invoked with an organization id whenever the act path needs a schedule store. Single-store embedders can ignore the id and return the same store every time.

A custom conversation-history source implements the read-only `MessageRetriever` trait over a database or a remote control plane. Implementing `get(session_id, message_id)` and `load(session_id)` is enough, because `load_filtered`, `load_filtered_history`, `load_page`, and `count` have working defaults built on `load`. `load_filtered_history` returns a `MessageHistory` whose `source_sequence` reports the highest persisted message-event sequence the load reflects; a turn can be pinned to a consistent snapshot of the event log with it. A new message is an `InputMessage` with role, content parts, optional controls, optional metadata, and tags, and no id or timestamp, since the storage layer generates those; `InputMessage::user(text)` covers the common case.

A full `Host` implementation exposes its stores through accessor methods: `harness_store(org_id)`, `agent_store(org_id)`, `session_store(org_id)`, `session_mutator(org_id)`, `provider_store(org_id)`, and `message_store()`. Two optional durability stores exist for distributed hosts. `durable_tool_result_store()` returns a `DurableToolResultStore` that claims and settles per-turn tool results, making tool execution idempotent across retries, and `partial_stream_store()` returns a `PartialStreamStore` that recovers an interrupted model response mid-stream. Both default to `None`: tools run fresh on every execution and there is no mid-stream recovery. A running `InProcessRuntime` exposes its canonical event log through `event_log()` for bounded per-session replay.

### The example hosts

Two standalone host applications show the framework API consumed the way an external Rust application would. The weekend concierge host is kept in the repository's root `examples/` folder on purpose: there it behaves like an external host application rather than a crate-local demo. Its entry point is `run_weekend_concierge_demo().await?` under `#[tokio::main]`.

````text
cargo run --manifest-path examples/weekend-concierge-host/Cargo.toml
cargo test --manifest-path examples/weekend-concierge-host/Cargo.toml
````

The framework CLI host runs against a real model by setting either `ANTHROPIC_API_KEY` or `OPENAI_API_KEY`; Anthropic wins if both are set, and any trailing words on the command line become the prompt.

````text
ANTHROPIC_API_KEY=... cargo run -p everruns-framework-cli-host
OPENAI_API_KEY=... cargo run -p everruns-framework-cli-host -- "Scale the api service to 4 replicas, then list the whole fleet."
````

### The distributed seam

The distributed seam is the Everruns Platform. There a server schedules work and workers execute phases and apply effects, while PostgreSQL stores workflow checkpoints and canonical events. The Platform runs the same `everruns-engine` turn state machine as the framework, adapted through the `everruns-durable` crate. This is a deployment boundary, not another configuration mode on `everruns::Engine`, and it cannot be selected through `LocalConfig`; of the three deployments this chapter opened with, it is the only one outside the framework engine. Remote applications use the Platform API or an SDK; product hosts compose the lower-level durable crates. Chapter 10 continues from that boundary.

*2026-09-17 03:24 - claude-fable-5.1*
