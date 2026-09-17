<!-- source: everruns/everruns @ 6cf2c15e5 (crate everruns 0.20.0). The working tree fast-forwarded to 7d9e07a0b (0.21.1) during extraction; identifiers and examples were audited against 6cf2c15e5. -->
# Workspaces and Background Work

Every agent you have built so far reads and writes files under `/workspace`, and every turn has run inside the request that started it. Neither is fixed. The same `/workspace` can be an in-memory scratch space, one shared host directory, a set of named roots, or an isolated Git worktree that survives a restart, and a session can hand work to a queue or a cron schedule and be woken when that work finishes.

## What a workspace is: Environments, heads, and the default workspace

````rust
use everruns::prelude::*;

let agent = Agent::builder()
    .instructions("Read brief.md and policy.md, then summarize both.")
    .model(Model::simulated("summary recorded"))
    .file("brief.md", "Investigate the supplied question.")
    .readonly_file("policy.md", "Never expose secrets.")
    .build()?;

let engine = Engine::new();
let session = engine.create(agent);
session.send_and_wait("Begin.").await?;
````

Nothing in this agent names a workspace, yet the session has one. With no workspace root and no local configuration, the framework gives every session a zero-config in-memory workspace, and nothing touches disk. The two seeded files appear inside the session at `/workspace/brief.md` and `/workspace/policy.md`. `.file(path, content)` seeds an editable UTF-8 text file and `.readonly_file(path, content)` seeds one the agent may read but not change. Paths may be relative (`brief.md`, `input/brief.md`) or absolute in the virtual namespace (`/workspace/weekend-brief.md`). For anything the two shorthands do not express, `.initial_file(InitialFile)` accepts a full starter-file description with path, content, encoding, and read-only flag. Starter files merge harness first, then agent, then session, and a host runtime can layer one more text file into a single session's workspace on top of that merged set.

A workspace is the logical project lineage the agent sees under `/workspace`. A workspace head is one named working copy of that workspace, provider-owned and mutable; you request one with a `WorkspaceHeadRequest` or the fluent `WorkspaceHeadBuilder` and access it as a `WorkspaceHeadResource` at one of two `WorkspaceHeadAccess` levels, `Isolated` or `Shared`, while `WorkspaceHeadStatus` reports its state. An `Agent` describes behavior and a `Session` owns conversation continuity. An `Environment` fixes the execution resources for that session, beginning with exactly one head. You build one with `EnvironmentBuilder` and attach a `WorkspaceBinding`; bindings are stored in an `EnvironmentBindingStore`, with `InMemoryEnvironmentBindingStore` as the provided in-memory implementation. These types are defined in `everruns-host`, the host layer beneath the framework. `Environment`, `EnvironmentBuilder`, `Workspace`, `WorkspaceBinding`, and the `WorkspaceHead*` types are re-exported from the `everruns` crate root; the binding store types stay in `everruns-host`.

````rust
let agent = Agent::builder()
    .instructions("Work only inside the configured workspace.")
    .model(Model::simulated("ready"))
    .workspace("./project")
    .build()?;
````

`.workspace(root)` points the agent at one real host directory. That directory becomes the shared default `/workspace` and the `session_file_system` capability is added automatically. Every session created from the agent binds to the same shared head before it executes. The shorthand is a first-class shared head, and it is exactly one head: a request for an additional or isolated head, or for a head with a base revision, is rejected as an invalid request, and the shorthand head cannot be archived or destroyed. The in-memory default provider is more permissive about access, accepting isolated or shared heads, but it rejects an external base. Under the `local` feature, an agent with no explicit root falls back to the `LocalConfig` workspace root you met in the previous chapter. A related pattern gives the model a known starting state without seeding through the builder: materialize fixture files into the host directory before the session runs, then point `.workspace(root)` at it, as the bashkit-repo-agent example does.

## Compute targets and containment

````text
cargo add everruns --features host-compute
````

````rust
use everruns::{Compute, ComputeSession, ExecRequest, ExecResult};
use everruns::{HostCompute, HostComputeSession};
````

The compute abstraction describes where agent commands execute. A `Compute` target has a `ComputeKind` and a set of `ComputeCapabilities`; you open a `ComputeSession` on it and submit an `ExecRequest`, which yields an `ExecResult`. `Containment`, `ContainmentLevel`, `NetworkPolicy`, and `Durability` complete the set; the whole group, with `ComputeError`, lives in the `compute` module of `everruns-host`.

Host compute is the one target that contains nothing. Commands run directly on the machine your process is already running on, with nothing between the command and the filesystem, which is why it is opt-in. The `everruns` feature is `host-compute`, defined as `host-compute = ["everruns-host/process"]`; it enables `HostCompute` and `HostComputeSession`, and inside `everruns-host` the same `process` feature also exposes `ProcessCommandExecutor`. Everything else in this chapter works on the default offline build.

## Workspace policy: presets, allow and deny scopes, and limits

### The read-only default and the read-write preset

````rust
use everruns::prelude::*;

let policy = WorkspacePolicy::read_only();
assert!(policy.permits_read("notes.txt"));
assert!(!policy.permits_write("notes.txt"));
assert!(!policy.permits_read(".env"));

let agent = Agent::builder()
    .instructions("Work only inside the configured workspace.")
    .model(Model::simulated("ready"))
    .workspace("./project")
    .workspace_policy(WorkspacePolicy::read_write())
    .build()?;
````

`WorkspacePolicy` is the portable security boundary for the files an in-process agent can see. You configure it entirely through `everruns`, with no host backends or runtime-owned blocklist to set up, and attach it with `.workspace_policy(policy)` on the agent builder.

You get the read-only policy without asking; `WorkspacePolicy::read_only()` and `Default` return the same layer. Ordinary files are readable and nothing is writable. Hidden paths, meaning dotfiles and dot-directories, are denied, with `.agents` kept readable so workspace instructions and skills still load. Recursive deletion is off. Two component lists apply at every depth regardless of allow scopes.

- Sensitive components, blocked for read and write by every policy: `.aws`, `.azure`, `.docker`, `.git`, `.git-credentials`, `.gnupg`, `.kube`, `.netrc`, `.npmrc`, `.pypirc`, `.ssh`, `credentials`, `credentials.json`, `gcloud`, plus `.env` and any `.env.*` name, matched without case sensitivity.
- Write-deny components, added by the read-write preset: `node_modules`, `target`, `dist`, `build`, `.next`, `.venv`, `venv`, `.tox`, `.gradle`.

`WorkspacePolicy::read_write()` enables ordinary reads and writes across the whole workspace in one call. It exposes no hidden or sensitive path, adds `.agents` to the write denials, leaves recursive deletion off, and keeps the dependency and build directories above non-writable wherever they appear.

### Custom scopes, composition, and checks

````rust
let policy = WorkspacePolicy::builder()
    .allow_read("/")
    .allow_write("generated")
    .deny_write("generated/locked")
    .deny_write_component("vendor")
    .allow_hidden(".github")
    .build()?;

assert!(policy.permits_write("generated/report.md"));
assert!(!policy.permits_write("generated/locked/result.txt"));
assert!(!policy.permits_write("src/vendor/generated.rs"));
assert!(policy.permits_read(".github/workflows/ci.yml"));
assert!(!policy.permits_read(".git/config"));
````

`WorkspacePolicy::builder()` starts deny-by-default with no readable or writable paths, and hidden and sensitive paths stay protected until you opt in. `.allow_read(path)` and `.allow_write(path)` open a scope. `.deny_read(path)` and `.deny_write(path)` close a subtree even when an allow scope covers it; deny wins over allow. `.deny_write_component(name)` rejects an exact directory or file name at every depth, which is how `vendor` above blocks `src/vendor/generated.rs` but not `src/vendorish/generated.rs`. `.allow_hidden(path)` admits dotfiles under one scope without exposing credential paths, so `.github` is safe to allow without also exposing `.git` or `.env`. `.allow_sensitive(path)` is the strongest opt-in and belongs on the narrowest scope you can write; it also permits hidden components within that scope, and it is exact, so allowing `.env.example` does not allow `.env` and allowing `.ssh/id_ed25519` does not allow `.ssh/config`. `.allow_recursive_delete(true)` turns on recursive directory deletion, off by default because deleting an allowed parent could remove denied or sensitive descendants that a lexical path check cannot detect; when you do opt in, descendants are inspected through the provider before deletion, so recursion never overrides a deny or a protected descendant.

`.build()?` is fallible. A scope containing traversal is rejected. A component deny with a slash in it fails with "one path component", and a blank scope fails with "must not be empty". Policies compose with `compose(other)`: a library can add restrictions to an application policy without ever broadening access, the result is the same in either composition order, and recursive deletion is permitted only if every layer allows it.

The boolean checks are `permits_read(path)` and `permits_write(path)`; one write decision covers create and delete as well as write. Invalid paths, including traversal attempts, are denied. `permits_read_traversal(path)` reports whether a directory may be traversed to reach a readable descendant, which filtered listings under narrow policies need: with `.allow_read("src/generated")`, traversal of `/workspace/src` is permitted while a read of it is not. `check_read` and `check_write` return a `WorkspacePolicyError` whose tool-safe message names the denied path as `/workspace/<path>`, and the associated function `WorkspacePolicy::validate_path` checks syntax alone with no access decision. Scope spelling follows four rules.

- Write scopes relative to `/workspace` (`src/lib.rs`), workspace-absolute (`/workspace/src/lib.rs`), or session-absolute (`/src/lib.rs`); all three identify the same file. Host absolute paths are provider-specific and are not portable scopes, so use `/workspace/...` in configuration and model instructions.
- Scopes are literal path prefixes, not globs. `generated` includes `generated/report.md` but not `other/generated/report.md`.
- Deny scopes and sensitive names compare ASCII letters without case sensitivity, so `Private`, `.ENV`, and `.Git` cannot bypass a deny. Allow scopes stay case-exact, so allowing `public` does not allow `Public/secret.txt`.
- Traversal (`..`), NUL bytes, and backslash-separated paths fail closed. Repeated slashes cannot bypass a deny scope.

A host runtime can also apply a policy that wraps whichever session filesystem the platform selected, independent of the storage backend. The `workspace_policy` example (`cargo run -p everruns --example workspace_policy`) exercises safe read and write scopes against the default restrictions and seeds trusted starter files, fully offline with no optional features.

## Workspace roots and virtual path resolution

````rust
use everruns_core::WorkspaceRootSet;

let roots = WorkspaceRootSet::new(
    "./app",
    [("backend".to_string(), "./services/backend")],
)?;

let primary = roots.parse_vfs_path("/workspace/src/lib.rs")?;
assert_eq!(primary.root_name, "workspace");
assert_eq!(primary.relative.as_relative(), "src/lib.rs");

let backend = roots.parse_vfs_path("/workspace/roots/backend/src/lib.rs")?;
assert_eq!(backend.root_name, "backend");

let host_path = roots.parse_host_scope("backend", Some("Cargo.toml"))?;
````

`WorkspaceRootSet` lives in `everruns-core` and is not re-exported at the `everruns` crate root, so import it from the core crate. `WorkspaceRootSet::from_primary(path)` gives an agent one primary directory, and `WorkspaceRootSet::new(primary, additional)` registers named host directories alongside it. Each additional root is exposed to the agent under the virtual mount `/workspace/roots/<name>`; the mount prefix is the constant `ADDITIONAL_ROOTS_MOUNT`. Additional roots are named mounts inside one selected head, not independent heads, and they have no fork or reopen lifecycle. The primary root is always named `workspace`, the constant `PRIMARY_WORKSPACE_ROOT_NAME`.

Construction validates and reports configuration errors. A root that does not exist or is not a directory fails to canonicalize. Two roots may not overlap, whether equal or one containing the other. Two additional roots may not share a name. A name is invalid if it is empty, `.`, `..`, `workspace`, or `roots`, or if it contains `/` or `\`. Resolution failures, by contrast, are tool errors the model can see and correct.

- `/workspace/<rel>`, the bare word `workspace`, `/<rel>`, and a bare relative path all resolve to the primary root.
- `/workspace/roots/<name>/<rel>` resolves to the named root. An unregistered name is an error.
- An absolute host path is mapped back to whichever registered root contains it; a host path outside every root is an error. `contains_host_path(path)` answers the same question as a boolean.
- Traversal (`..`) is rejected. Leading and trailing whitespace is trimmed, and empty and `.` segments are dropped.

`parse_vfs_path(input)` returns a `ResolvedPath` with `root_name` and `relative`; `relative.as_relative()` is the joined segments and `to_session_path()` is the `/`-prefixed session form. `parse_host_scope(root, relative)` translates the other way into a canonical host path. It rejects anything that escapes the root, including symlink escapes, and resolves not-yet-existing files through their parent directory. The workspace runtime repeats its containment checks on every filesystem operation, so a symlink introduced after configuration is still blocked. `set_primary_host_root(path)` re-points the primary at runtime without touching additional roots, and a rejected re-point leaves the registered set unchanged. `primary_host_root()` reads the current primary, and `spawn_cwd()` returns it canonicalized as the working directory for spawned processes.

## The session filesystem contract

### Implementing the trait

````rust
#[async_trait]
pub trait SessionFileSystem: Send + Sync {
    fn is_mount_resolver(&self) -> bool;
    async fn read_file(&self, session_id: SessionId, path: &str) -> Result<Option<SessionFile>>;
    async fn write_file(&self, session_id: SessionId, path: &str, content: &str, encoding: &str) -> Result<SessionFile>;
    async fn delete_file(&self, session_id: SessionId, path: &str, recursive: bool) -> Result<bool>;
    async fn list_directory(&self, session_id: SessionId, path: &str) -> Result<Vec<FileInfo>>;
    async fn stat_file(&self, session_id: SessionId, path: &str) -> Result<Option<FileStat>>;
    async fn create_directory(&self, session_id: SessionId, path: &str) -> Result<FileInfo>;
    async fn grep_files(&self, session_id: SessionId, pattern: &str, path_pattern: Option<&str>) -> Result<Vec<GrepMatch>>;
    // Provided with defaults: display_root, display_path, resolve_path,
    // write_file_if_content_matches, grep_files_with_options, seed_initial_file
}
````

Any storage can back an agent's workspace by implementing this one async trait: a database in production, an in-memory filesystem for tests, real disk, or object storage. The host crate ships `RealDiskFileStore` for real disk and a `multi_root_file_system` helper that spans several directories; `RealDiskSessionFileSystemFactory` creates a real-disk filesystem per session.

Whatever the storage, the agent sees a stable `/workspace` root. `display_root()` defaults to that prefix, and `resolve_path(input)` turns any accepted spelling into an absolute path inside the namespace, with relative inputs resolved against the filesystem's current directory. That is how a shell seeds its working directory so shell and file tools share one identity. A provider that accepts extra aliases must use the same contained mapping in `resolve_path` and in its I/O methods; an alias must never resolve to one workspace path during resolution and a different storage object during the operation that follows.

- `read_file` returns `None` for a missing file rather than an error. `write_file` takes content and an encoding. `delete_file` takes an explicit `recursive` flag.
- `write_file_if_content_matches(session_id, path, expected_content, expected_encoding, content, encoding)` is the compare-and-set write. A missing file or a directory returns `Ok(None)`, as does a content or encoding mismatch, and nothing is written. Transactional backends should override it with an atomic update. A shared real-disk head does not become isolated: compare-and-set reports stale-content conflicts within the host process, and provider status reports Git conflict and dirty metadata. Coordinate other writers at the application or provider layer.
- `grep_files` searches contents with Rust regex syntax, optionally filtered by `path_pattern`; basename-only globs match at any depth and non-glob filters use substring matching. `grep_files_with_options(session_id, pattern, &GrepOptions)` adds `before_context` and `after_context` lines and returns a `GrepSearchResult` bounded by `GREP_MAX_CONTEXT_LINES` and `GREP_MAX_RETURN_BYTES`. A store that implements only the zero-context form reports an error when context is requested.
- `seed_initial_file(session_id, &InitialFile)` writes a starter file. The default delegates to `write_file` and returns an error for a read-only starter file, which needs a filesystem-specific seed implementation.

### Scoping, errors, and factories

When a session is attached to a shared workspace whose id differs from the session id, `WorkspaceScopedFileSystem::wrap(inner, workspace_id)` pins every operation to the workspace's key and ignores the per-call `session_id`. For the default one-to-one session the key equals the session id and the wrapper is a transparent pass-through. Filesystems are mounted into the agent's view through `MountFs` with a `DisplayPolicy`, so `/workspace` is a mount plus a current directory and no store prefixes its own paths.

Failures are classified so the runtime can decide whether the agent can correct them. `classify_fs_error` walks an error's source chain and returns a `FileSystemErrorClass` of `NotFound`, `ReadOnly`, `IsADirectory`, `NotADirectory`, `NotEmpty`, or `Other`; a typed `FileSystemError` is found even when wrapped inside another error.

Which filesystem each session sees is chosen by a `SessionFileSystemFactory`, given a `SessionFileSystemFactoryContext`. `FixedSessionFileSystemFactory` pins one filesystem for every session, and `DisabledSessionFileSystemFactory` turns file access off. The older decorators `ApprovalGatingFileStore` (with `FileApprovalGate`), `PolicyFileStore`, and `WriteBlocklistFileStore` still exist but are deprecated. Two host-level operations belong to the advanced host layer covered in the next chapter: the host session builder scopes a session to an explicit `WorkspaceId` with `.workspace(WorkspaceId)`, deriving the id from the session id when unset, and the host runtime reads a session file with `read_file(session_id, path)`.

## Git workspaces, heads, and binding sessions to Environments

### Creating and managing Git heads

````text
cargo run -p everruns --features local --example workspace_heads -- /path/to/repo /path/to/state
````

````rust
use std::sync::Arc;
use everruns::prelude::*;
use everruns::{LocalGitWorkspaceProvider, Workspace};

let provider = Arc::new(LocalGitWorkspaceProvider::new(state.join("workspace-provider"))?);
let workspace = Workspace::open(provider.clone(), repository.to_string_lossy()).await?;
let head = workspace.head("example").create().await?;
````

A workspace provider is a backend that implements the workspace provider trait, through which the framework opens workspaces and manages the heads inside them, from creation and reopen through the rest of their lifecycle. The framework registers two providers itself, the default memory provider `everruns.framework.memory.v1` and the directory shorthand provider `everruns.framework.directory.v1`. With the `local` feature, `LocalGitWorkspaceProvider` (`everruns.local-git-worktree.v1`) adds a third, in which every writable head is an isolated Git worktree.

`LocalGitWorkspaceProvider::new(state_root)?` opens or creates provider state under a directory; its `workspaces`, `heads`, and `worktrees` children are created on demand. Wrap it in an `Arc` so the same provider serves both the `Workspace` and the `Agent`. `Workspace::open(provider, path)` accepts any path inside a Git checkout and resolves the repository root itself; the workspace name defaults to the repository directory name. `workspace.head("example").create()` makes a fresh worktree on a new branch named `everruns/<head id>`, based on `HEAD`; calling `.from_revision("main")` before `.create()` selects another base.

- Heads are isolated by default (`WorkspaceHeadAccess::Isolated`). A file written in one head is invisible in another, and binding the same isolated head to a second session is rejected. `workspace.head("shared").shared().create()` opts into a shared mutable head.
- `head.fork("name").await` creates a new isolated head from the original's committed state.
- Checkpoint records the head's current Git revision and head name. Status reports whether the head has uncommitted changes, whether it has merge conflicts, whether it is archived, and its branch name; a diff summary reports `changed` and `conflicted`.
- Archive blocks future reopen while keeping the worktree on disk, and it does not revoke filesystem handles a running session already holds. Destroy removes the worktree; the Git branch stays durable in the repository.
- `workspace.reopen(head.binding())` reopens a head from its persisted binding and refreshes its Git revision. A destroyed head reopens as `NotFound` and an archived one as `Archived`.

Head names are 1 to 256 non-blank characters. Base revisions are non-empty, at most 512 characters, never start with `-`, and never contain NUL. Nothing is cleaned up implicitly: dropping a provider, workspace, head, session, or agent never deletes a worktree or branch. Repository Git hooks never execute when the provider inspects or checks out a repository, and Git never prompts for credentials. The state root and its children must be real directories rather than symlinks, and metadata files larger than 1 MiB are rejected. Repository paths must be valid UTF-8 for durable resume.

### Binding a session to an Environment

````rust
let agent = Agent::builder()
    .instructions("Edit files in the checked-out head and report what changed.")
    .model(Model::simulated("changes recorded"))
    .workspace_policy(WorkspacePolicy::read_write())
    .workspace_provider(provider)
    .local(LocalConfig::new(state.join("data")))
    .build()?;

let engine = Engine::new();
let session = engine.create(agent).workspace(head).start().await?;
assert!(session.workspace_head().is_some());
````

`.workspace_provider(provider)` registers the provider on the agent so a freshly constructed agent can resume persisted, provider-owned heads after a restart; starting a session from a live environment also retains its provider for the agent's lifetime. Registering two providers with the same id, or one that collides with the default provider, fails at build time with `BuildError::DuplicateWorkspaceProvider { id }`.

`engine.create(agent).workspace(head)` binds the new session to that head and returns an `EnvironmentSessionBuilder`; `.start().await?` persists the opaque head binding before any runtime can execute. It is shorthand for `engine.create(agent).environment(Environment::new(head))`, the form you use when the Environment also needs typed extensions. A session can never switch heads after it starts.

For an agent with a default workspace and no explicit head, `session.start().await?` selects and persists the default head without running a turn. Sending a message or inspecting the session performs the same one-time selection automatically, and starting again once bound is a no-op. `session.workspace_head()` returns `None` before start and the selected head afterward. `session.environment_extension::<T>()` resolves a typed resource attached to the Environment: the Environment is its head plus an open type-keyed extension boundary for future compute or network resources, attached through `EnvironmentBuilder::workspace_extension`, whose constructor receives the same head the file tools will use.

On resume, the engine asks the recorded provider to reopen the recorded workspace and head. Directory workspace and head ids derive deterministically from the canonical path, so a persisted session comes back against the same directory across restarts. Resume never substitutes an empty or different head. It fails with `ResumeError::WorkspaceBindingCorrupt` when the binding cannot be decoded, `ResumeError::WorkspaceProviderUnavailable { provider_id }` when the recorded provider is not registered, `ResumeError::WorkspaceUnavailable` when the head cannot be opened, and `ResumeError::WorkspaceMismatch` when the provider returns a different workspace or head than was recorded.

Binding errors are the non-exhaustive `SessionEnvironmentError`. `AlreadyStarted` means the session already ran, `AlreadyBound` that it is bound to a different head, `ProviderConflict` that the head's provider differs from one the session already retained, `Unavailable` that backends cannot be reached, and `Workspace(WorkspaceError)` that the provider reported a workspace error.

## Workspace security rules

````rust
let policy = WorkspacePolicy::builder()
    .allow_read("/")
    .allow_write("generated")
    .allow_hidden(".agents")
    .build()?;

let agent = Agent::builder()
    .instructions("Follow the workspace instructions and write results under generated/.")
    .model(Model::simulated("done"))
    .workspace("./project")
    .workspace_policy(policy)
    .readonly_file("policy.md", "Never expose secrets.")
    .build()?;
````

A custom builder does not inherit the default `.agents` exception. The policy above restores workspace-provided instructions and skills by pairing a readable scope with a narrow `.allow_hidden(".agents")`. The read-only starter file is placed at `/workspace/policy.md` even though the policy grants no write there, because trusted starter files are installed before model access is enforced. Every later operation on that file still goes through the policy, and seeding a hidden file does not expose it.

Workspace roots belong in trusted application configuration; model output and untrusted request fields must never select executable paths or host directories. The policy governs only capabilities that use the framework's session filesystem, so a custom tool that calls `std::fs` directly or launches a shell does not pass through it, and neither does one that uses another storage API; apply equivalent restrictions to those tools or run them in a sandbox. `WorkspacePolicy` is path authorization, not a compute sandbox: a malicious process running as the same operating system user can race a path check against the filesystem call, and the policy is no substitute for a process boundary.

Inside that boundary, discovery is treated like access. Directory listings and content search are enforced at the same point as direct reads, so a denied file is never opened and its name never surfaces in a result, not even as a match count or a byte total.

## Session-owned background work: queues, tasks, and wakes

### Submitting work from the session side

````text
cargo run -p everruns --example session_work
````

````rust
use everruns::prelude::*;
use everruns::work::{TaskRequest, WakePolicy, WorkQueue};
use serde_json::json;

let queue = WorkQueue::in_memory();
let session_work = session.work(&queue);

let task = session_work
    .submit(
        TaskRequest::new("thumbnail", json!({ "image": "cover.png" }))
            .idempotency_key("thumbnail-cover")
            .wake_policy(WakePolicy::OnCompletion),
    )
    .await?;
println!("submitted {} in state {:?}", task.id, task.state);
````

Nothing in this snippet imports a runtime registry or a platform store, and nothing names a task-kind constant. `everruns::work` is built so an application can request and handle session-owned work without any of them. `WorkQueue::in_memory()` creates a process-local queue that needs no database and no network, and `session.work(&queue)` scopes it to the session and returns a `SessionWork` handle, so every task, read, cancellation, and wake through that handle is owned by that session; `queue.for_session(id)` does the same from a session id string. When work must survive a restart, `WorkQueue::with_backend(Arc<dyn WorkBackend>)` plugs in a host-owned durable persistence and leasing backend, and application code keeps the same API.

The request itself is two values: an application-defined routing kind, which is any string, and arbitrary JSON input, stored unchanged and delivered to the worker. Unmodified, `TaskRequest::new(kind, input)` runs immediately (`WorkSchedule::Immediate`) with no idempotency key, and never wakes the session (`WakePolicy::Never`).

Each of those defaults has a builder call. `.schedule(WorkSchedule::At(SystemTime))` holds a one-shot task until no earlier than a wall-clock time. `.idempotency_key(key)` deduplicates within the session: repeating the same request with the same key returns the original task, and reusing the key for a different request fails with `WorkError::IdempotencyConflict { key }`. `.wake_policy(WakePolicy::OnCompletion)` creates one leased wake when the task first reaches a terminal state, and cancellation counts as terminal.

Once submitted, a task is reached through the same `SessionWork` handle, which offers `submit`, `task(task_id)`, `cancel(task_id)`, and `wake(WakeRequest::new(json).idempotency_key(k))`. Tasks owned by other sessions read as `None`. Pending work cancels immediately; running work gets `cancel_requested_at` set, and its handler must stop cooperatively and finish with `TaskOutcome::Canceled`. Repeating a cancel is safe. A direct wake records an immediate wake for the session with an arbitrary payload, and task keys and direct-wake keys are kept in separate session-scoped namespaces. Each method has an `_at` twin (`submit_at`, `cancel_at`, `wake_at`) that takes an explicit `SystemTime` for deterministic tests.

What comes back is a `Task` with `id`, `session_id`, `kind`, `input`, `state`, `cancel_requested_at`, `idempotency_key`, `scheduled_at`, `created_at`, `attempts`, `finished_at`, and `outcome`. `TaskState` is `Pending`, `Running`, `Succeeded`, `Failed`, or `Canceled`; `is_terminal()` is true for the last three.

### Handling work from the worker side

````rust
use std::time::Duration;
use everruns::work::{TaskOutcome, WakeReason};

for delivery in queue.claim_due(Duration::from_secs(30), 16).await? {
    let output = match delivery.task.kind.as_str() {
        "thumbnail" => json!({ "path": "cover-thumb.png" }),
        kind => return Err(format!("no handler for {kind}").into()),
    };
    queue.finish(&delivery, TaskOutcome::success(output)).await?;
}

for delivery in queue.claim_wakes(Duration::from_secs(30), 16).await? {
    if let WakeReason::TaskFinished { outcome, .. } = &delivery.wake.reason {
        session.run(format!("Background task completed: {outcome:?}")).await?;
    }
    queue.acknowledge_wake(&delivery).await?;
}
````

The host owns polling and execution. `queue.claim_due(lease_for, limit)` claims a batch of due or lease-expired tasks, here with a 30 second lease and up to 16 deliveries; `claim_due_at(now, lease, limit)` is the deterministic form. Route on `delivery.task.kind` and check for cancellation before side effects. Settle with `queue.finish(&delivery, outcome)` using `TaskOutcome::success(json)`, `TaskOutcome::success_with_summary(str, json)`, `TaskOutcome::failure(str)`, or `TaskOutcome::Canceled`. `claim_wakes(lease_for, limit)` has the same shape and returns `WakeDelivery` values, and `acknowledge_wake(&delivery)` marks one handled idempotently. A wake's `reason` is either `WakeReason::Requested { payload }` or `WakeReason::TaskFinished { task_id, state, outcome }`, with the outcome copied in so no second read is needed. Resuming the agent is an ordinary message derived from the outcome.

Work and wake delivery are at least once. An unacknowledged delivery becomes claimable again after its lease expires, with a new `attempt` number, so deduplicate external side effects on the stable `task.id` or the idempotency key. Every delivery also has a per-attempt `lease_token()`. A superseded worker that tries to settle or acknowledge newer work gets `WorkError::StaleDelivery { id }`. Repeating a settlement with the same token and the same outcome succeeds; the same token with a different outcome is `WorkError::InvalidTransition`. Tokens are redacted from `Debug` output. The rest of the non-exhaustive `WorkError` enum is `InvalidRequest { field, message }`, which covers a blank session id, task id, kind, or idempotency key as well as a zero lease duration or batch limit, plus `TaskNotFound { task_id }` and `Backend { message }`.

In-memory state survives replacing the queue only while the same `Arc<InMemoryWorkBackend>` is retained, and it does not survive a process restart. Separately created `WorkQueue::in_memory()` instances are isolated. The in-memory backend honors the same contract as a durable one, which makes it suitable for deterministic tests, but it retains payloads until dropped and applies no admission quota, so hosted providers must apply tenant authorization, payload limits, quotas, and retention at their own boundary.

A custom durable backend implements `WorkBackend` with `submit`, `task`, `cancel`, `claim_due`, `finish`, `request_wake`, `claim_wakes`, and `acknowledge_wake`, expressed only in application-level tasks and wakes. The guarantees above are the backend's to uphold: `submit` and `request_wake` enforce the session-scoped idempotency, and `finish` and `acknowledge_wake` do the token fencing. `Task::pending`, `TaskDelivery::from_claim`, `WakeDelivery::from_claim`, `SessionWake::requested`, and `SessionWake::task_finished` are the constructors for backend authors, and tokens must be unique per attempt and unguessable across a trust boundary. Durable scheduling, retention, distributed polling, and multi-host coordination belong in that backend; the offline default does not enable them.

### Detaching work from a tool call

A tool can detach long-running work: spawn it and return at once with text such as "Started <task id>. End this turn; the host will wake you when the checks finish.", so the foreground turn can end. Later the host sends a follow-up such as "[automatic] Background task <id> <status>." followed by the result body through `session.send_and_wait`, and the agent reviews the outcome. The `github_monitor` example does this for a GitHub pull request's checks. `cargo run -p everruns --features openai --example github_monitor -- --simulate` skips GitHub access but still uses the configured model, and `-- --live OWNER/REPO PR_NUMBER` requires an authenticated GitHub CLI. Selecting the mode changes no agent code.

## Session tasks, the task registry, and background tools

````rust
use everruns::local::LocalSessionTaskRegistry;
use everruns_core::SessionTaskRegistry;

let registry = LocalSessionTaskRegistry::new(db)?;

let task = registry.get(session_id, &task_id).await?;
let all_tasks = registry.list(session_id, None).await?;
registry.request_cancel(session_id, &task_id).await?;
let history = registry.list_messages(session_id, &task_id, Some(50), None).await?;
````

Separate from the work queue, the core session-task model tracks tasks of several kinds, each named by a constant: `TASK_KIND_SUBAGENT`, `TASK_KIND_BACKGROUND_TOOL`, `TASK_KIND_AGENT_HANDOFF`, `TASK_KIND_EXTERNAL_AGENT`, `TASK_KIND_MONITOR`, and `TASK_KIND_SESSION`. A `SessionTask` has a `TaskExecutor`, `TaskProgress`, `TaskArtifact` values, a `TaskWakePolicy`, an optional `TaskInputRequest`, and a durable message channel of `TaskMessage` values. `SessionTaskRegistry` is the trait every store implements; it is defined in `everruns-core`, and you import it to call the registry methods below.

Under `local`, `LocalSessionTaskRegistry::new(db)` constructs a SQLite-backed registry over an existing `SqliteDb` handle, the same local database the previous chapter introduced, and creates and migrates the schema on open. It exists so tasks survive a restart: a freshly spawned process reopens the database file and picks up its tasks where they were.

- `create(CreateSessionTask)` timestamps the task and writes it durably before returning. Supplying your own `id` makes creation idempotent: the same id in the same session returns the existing task, and the same id under a different session is rejected so a caller cannot alias another session's task.
- `update(session_id, task_id, SessionTaskUpdate)` applies a state or field change and returns the updated task, or `None` if it does not exist.
- `get(session_id, task_id)` fetches one task. Tasks from other sessions are invisible even with a known id.
- `list(session_id, filter)` returns a session's tasks in stable creation order. `SessionTaskFilter` narrows by `kind`, by `state`, or both, with the predicates pushed into indexed SQL.
- `request_cancel(session_id, task_id)` records intent in `cancel_requested_at` without forcibly changing the state.
- `record_message(session_id, task_id, NewTaskMessage)` appends an inbound or outbound message and returns the stored `TaskMessage` with a generated id and timestamp. Setting `expected_attempt` on the message fences out stale executors, so a write from a superseded attempt is rejected. An inbound message whose `in_reply_to` matches the task's pending input request id clears the request and returns the task to `SessionTaskState::Running`; outbound messages never resume a task.
- `list_messages(session_id, task_id, limit, after_id)` reads history oldest-first.

The host runtime builder registers a registry with `with_session_task_registry(registry)` and needs no full backend bundle to do it. Once a registry is configured, a background task transition can wake a running turn mid-flight: an `ObservingTaskRegistry` wraps the inner registry and feeds a `SessionWakeQueue`, the wake is injected as a user message before the next model call, and each wake is delivered once, either mid-turn or on the next turn's first drain.

A tool declares detached execution through the `supports_background` hint on its definition. The `BackgroundExecutableTool` contract streams `BackgroundProgress` and a `BackgroundOutcome` back into the session through a `BackgroundEventSink`. Nested subagent sessions spawn under a `SubagentNestingPolicy` with claim tracking in a `SubagentSpawnStore`; core owns the host-neutral `SubagentSessionDelegate` interface and a host adapter supplies the implementation.

## Schedules, the schedule store, and cron

### Creating schedules

````rust
use chrono::{Duration, Utc};
use everruns::local::LocalScheduleStore;
use everruns_core::session_services::SessionScheduleStore;

let store = LocalScheduleStore::new(db, org_id, owner_principal_id)?;

let weekday_digest = store
    .create_schedule(
        session_id,
        "Summarize the pull requests opened since yesterday.".to_string(),
        Some("0 9 * * 1-5".to_string()),
        None,
        "America/New_York".to_string(),
    )
    .await?;

let reminder = store
    .create_schedule(
        session_id,
        "Check whether the deploy finished.".to_string(),
        None,
        Some(Utc::now() + Duration::minutes(30)),
        "UTC".to_string(),
    )
    .await?;
````

With the `local` feature, agent tasks can be scheduled with cron expressions and timezone-aware timing; the feature pulls in the `cron`, `chrono`, and `chrono-tz` dependencies. The Schedules capability, id `session_schedule` with 3 tools, lets the agent create and manage its own schedules and contributes the `schedules` feature. Those tools work through a `SessionScheduleStore`, a trait in `everruns-core` that you can implement yourself; `create_schedule`, `cancel_schedule`, `list_schedules`, and the count methods below are trait methods, so the trait must be in scope to call them. `LocalScheduleStore` is the SQLite one that survives restarts: `LocalScheduleStore::new(db, org_id, owner_principal_id)?` opens and migrates it over an existing `SqliteDb`, scoped to an organization id, with an owner principal stamped on every schedule it creates.

`create_schedule(session_id, description, cron_expression, scheduled_at, timezone)` is the standard method, and the description becomes the message delivered to the session when the schedule fires. A one-shot schedule passes `scheduled_at` in UTC with `None` for the cron expression; it fires once and is then disabled. A recurring schedule passes `Some(cron)`, and its next trigger time is computed immediately at creation and advanced after each delivery. `create_schedule_with_metadata(..., metadata: Value)` additionally attaches a JSON bag for fields the core `SessionSchedule` type does not have, such as name, color, kind, command, model, and isolated; the standard method stores an empty bag. `get_metadata(schedule_id)` reads it back and returns `None` for a schedule outside this organization scope.

`cancel_schedule(session_id, schedule_id)` disables the schedule and preserves its metadata bag; a missing id reports "schedule not found". `list_schedules(session_id)` returns every schedule, enabled or disabled, in creation order. `count_active_schedules(session_id)` and `count_active_org_schedules()` count enabled schedules for one session and for the whole organization.

- Write cron in the standard 5-field form (minute hour day month weekday), or in 6-field or 7-field form with seconds first and an optional trailing year. A 5-field expression is normalized to `"0 {fields} *"`. Any other field count is a configuration error.
- An expression that can never fire again is rejected with "cron expression has no future occurrence".
- `timezone` is an IANA name such as `America/New_York`. Occurrences are computed in that zone so recurring schedules fire at the right local wall-clock time, and an invalid name is a configuration error.

Three limits apply. Cron expressions that fire too frequently are rejected; the minimum interval comes from `SESSION_SCHEDULE_MIN_INTERVAL_SECONDS` (positive integer seconds), falling back to `DEFAULT_MIN_INTERVAL_SECONDS`. Active schedules per session are capped by the fixed constant `MAX_ACTIVE_SCHEDULES_PER_SESSION`, which is not tunable; the error reads "Maximum N active schedules per session. Cancel an existing schedule first." Active schedules per organization are capped by `RESOURCE_LIMIT_MAX_SESSION_SCHEDULES_PER_ORG` (positive integer), falling back to `DEFAULT_MAX_SCHEDULES_PER_ORG`, and non-positive or unparseable values are ignored. `create_schedule_enforcing_limits` checks both caps and the minimum interval in one database call before inserting, and backends with shared mutable state must make that check-and-create atomic.

### Running due schedules

````rust
use std::time::Duration;
use everruns::local::{LocalScheduleRunner, LocalScheduleRunnerConfig};

// Production defaults: 1 second poll, 30 second claim timeout, batch of 32.
let handle = backends.start_schedule_runner(session_runner.clone())?;

// Or tune it.
let config = LocalScheduleRunnerConfig {
    poll_interval: Duration::from_secs(2),
    claim_timeout: Duration::from_secs(60),
    batch_size: 8,
};
let handle = LocalScheduleRunner::new(backends.schedule_store()?, session_runner)
    .with_config(config)
    .start()?;

// At host shutdown.
handle.shutdown().await?;
````

The schedule runner is an in-process executor that delivers due schedules from the local store to agent sessions. The host starts it explicitly; nothing starts it for you. `backends.start_schedule_runner(runner)?` on the `LocalBackends` bundle from the previous chapter uses production defaults, and `start_schedule_runner_with_config(runner, cfg)?` overrides them. `LocalScheduleRunner::new(store, session_runner).with_config(cfg).start()?` is the hand-built form; `backends.schedule_store()?` returns the concrete `LocalScheduleStore` when you need its metadata methods. The session runner is any `LocalSessionRunner` implementation; the runner calls `routable_session_ids()` on it and delivers with `send_message(session_id, &description)`. Starting spawns a background polling task and returns a `LocalScheduleRunnerHandle`. Retain it for the host lifetime, because dropping it stops the runner.

`LocalScheduleRunnerConfig` has the three fields whose production defaults appear in the comment above. `poll_interval` is how often the store is checked for due schedules. `claim_timeout` is how long an unfinished claim stays exclusive before another runner may recover it, and a failed delivery waits the same period before it becomes claimable again. `batch_size` is the maximum schedules claimed per poll. Zero values are rejected at start, and `batch_size` may not exceed 1,000.

A due schedule is delivered by sending its description as a message to its session. On success a recurring schedule advances to its next cron occurrence and a one-shot is disabled, and its trigger count and last-triggered time are updated. Long deliveries send a claim heartbeat every third of the claim timeout so no other runner can claim them, and a lost claim aborts the delivery with an error. When a delivery fails, the schedule is retained and the error text is stored, truncated to 4096 bytes, readable through `last_delivery_error(schedule_id)`, and cleared on the next successful delivery. The runner only claims schedules for sessions it can route to; one that reports no routable sessions claims nothing.

`handle.shutdown().await` stops claiming and waits for the in-flight delivery, while `handle.abort()` stops immediately and leaves any unfinished claim to time out. Start, stop, claims, deliveries, and failures are logged through structured tracing, tagged with the runner id and the schedule and session ids, and a failed poll is logged and the loop continues.

A host runtime can inject a per-organization schedule store factory with `with_schedule_store_factory(factory)`. The same store backs the `usage_limit_auto_continue` capability, which schedules a continuation after a provider usage limit resets.

## Wake routing for host-driven sessions

````rust
use std::sync::Arc;
use everruns::local::{HostRoutedRunner, WakeRoutes};

let routes = WakeRoutes::new();
let runner = HostRoutedRunner::new(inner_runner, routes.clone());
let handle = backends.start_schedule_runner(Arc::new(runner))?;

let session_id = session.session_id();
let mut wakes = routes.register(session_id).await;
while let Some(message) = wakes.recv().await {
    let turn = session.send_and_wait(message).await?;
    println!("{}", turn.response);
}
routes.unregister(session_id);
````

When a host application drives a session in its own loop, a background completion delivered underneath it would mean two turns on one session at once. `HostRoutedRunner::new(inner, routes)` wraps any `LocalSessionRunner` and implements the trait itself, so it can be used anywhere a runner is accepted, including the schedule runner above. If the session has a registered route, the wake message is sent to the host's channel. If there is no route, as with a child or subagent session nobody is watching, the inner runner runs the turn synchronously. Both types require the `local` feature and live in `everruns::local`, alongside `LocalScheduleRunner`, `LocalScheduleStore`, and `LocalSessionTaskRegistry`.

`WakeRoutes::new()` creates the registry. It is cloneable and clones share state, so one clone goes to the runner and one stays with the host. `routes.register(session_id).await` marks a session host-driven and returns an `UnboundedReceiver<String>` on which the loop receives wake messages as plain strings such as "Background run completed."; the host turns each into an ordinary send. `routes.unregister(session_id)` removes the route. `routes.live_sessions()` lists the sessions currently claimed by a live loop, and the routed runner reports those as routable alongside whatever the inner runner reports.

Re-registering a session replaces its route; the newest loop wins, which is the right behavior for a reconnecting host. `register` is async because it waits until any in-flight synchronous fallback turn for that same session has finished before the host becomes live, so two loops never run one session. Other sessions' turns never block it, and a turn that wakes its own session again does not deadlock. If a host crashes or closes and drops its receiver, the stale route is pruned on the next wake, and that wake takes the no-route path instead of being lost. Immediate background completions are not retried.

*2026-09-17 03:24 - claude-fable-5.1*
