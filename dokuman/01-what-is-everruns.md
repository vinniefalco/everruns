<!-- source: everruns/everruns @ 6cf2c15e5 (crate everruns 0.20.0). The working tree fast-forwarded to 7d9e07a0b (0.21.1) during extraction; identifiers and examples were audited against 6cf2c15e5. -->
# What Everruns Is

Everruns is a Rust framework for building AI agents inside the program you are already writing. You add one crate, describe an agent as a plain value, pass it to an engine, and run conversations with it from ordinary Rust code. Both the agent and the engine are values your application owns, and that ownership is the idea the rest of the vocabulary grows out of.

## What the Everruns Framework is

The Everruns Framework is the `everruns` crate, the application-facing library of the Everruns project. With it you describe agents, attach models and tools, run multi-turn sessions, observe events, and embed agent execution directly in a Rust process. Here is a complete agent description. It needs two imports and nothing else.

````rust
use everruns::{Agent, Model};

let agent = Agent::builder()
    .instructions("Remember the conversation.")
    .model(Model::simulated("Acknowledged."))
    .build()?;
````

`Agent` is the type that describes what an agent does. `Model` names what runs the reasoning, and `Model::simulated` is a stand-in model with a scripted reply. The `build()` step validates the description and returns a `Result`, so a mistake in the description surfaces as an error at build time rather than mid-conversation. Nothing here mentions a harness, a hosted agent record, an id, a timestamp, a registry, or a host composition; the harness and the host are defined later in this chapter.

The crate exists so that Rust application authors can describe and run agents without first becoming execution-host implementers. You program against value-first APIs: you build values that state what you want, and the crate maps those values onto neutral core and provider contracts and a canonical in-process host. Host construction and platform record-keeping never appear in your code.

Everything runs inside your own Rust process, which makes the Framework a fit for libraries, command-line programs, desktop applications, services, and tests that want agent behavior as an ordinary application concern rather than as separate infrastructure. From application code you configure instructions, models, providers, tools, files, sessions, observation, and controlled extension points, and you can start with one agent and later move to a custom runtime without replacing the programming model you started with.

The runtime has four explicit pieces with a documented data flow.

````text
Agent + Provider + Tools  ->  Engine  ->  Session  ->  Turns and Events
````

A provider here is the vendor driver that runs a model; chapter 04 covers providers.

## Agent, Engine, Session, and Turn

The four nouns from the data flow are four types. Here they are working together in one program.

````rust
use everruns::{Agent, Engine, Model};

let agent = Agent::builder()
    .instructions("Remember the conversation.")
    .model(Model::simulated("Acknowledged."))
    .build()?;

let engine = Engine::new();
let session = engine.create(agent.clone());

let first = session.send_and_wait("hi").await?;
let second = session.send_and_wait("continue").await?;
println!("{}", second.response);
````

### Agent and Engine

An `Agent` describes behavior: instructions, model and provider, tools, capabilities (defined in the next section), files, and lifecycle hooks. You compose it with the fluent builder you saw above, setting a name, instructions, a provider, a model, and tools, then finish with `build()`. Once built, the value is immutable, so changing behavior means building a new `Agent`; there is no method that mutates an existing one. That immutability is what lets one `Agent` serve many conversations. Your application owns the value, passes it to an engine your application also owns, and that engine creates independent sessions from it while the `Agent` itself holds none of the conversation state. An `Agent` built with only instructions and a simulated model is already enough to run conversations.

An `Engine` is created with `Engine::new()` and takes no arguments. It owns runtime resources and the session catalog, and it is the single process-local owner of agent snapshots, session identity, backends, history, and resume authority. The same `Engine` value creates new sessions with `create` and resumes existing ones with `resume`, independently of any single agent instance, so keep it alive for as long as you want to be able to resume its sessions; drop it and that ability goes with it.

`Engine::new()` and `InMemoryEngine::new()` are interchangeable; both create and resume sessions and both are process-local. The default engine keeps everything in memory and touches no database, and a session it creates is hosted entirely by your application, with no server or worker service in the path. It is also volatile: it can resume a session even after your code drops the session value, but when the process exits the session is gone. Sessions that must outlive the process need the opt-in `local` persistence feature (chapter 08).

### Session and Turn

A `Session` is an isolated, multi-turn conversation bound to the Engine that created it. Everything you do to a conversation goes through the session, whether that is sending a message, steering the model, observing events, cancelling in-flight work, inspecting context, or reading history. Each session is independent of every other session created from the same `Agent`, and history accumulates across turns inside one session. The first `send` or `inspect` on a session materializes an isolated in-process runtime; later operations reuse it. In the example above, when the second `send_and_wait` runs the model sees the first exchange, which is why the instructions say "Remember the conversation." and why a fresh session started from the same `Agent` would not see it.

A `Turn` is the record of one run: the response, the status, the iteration count, and the tool-call count. You read `turn.response` for the model's reply, check the status to confirm the turn stopped successfully, and read `turn.tool_calls` to see which tools ran. The example prints `second.response` and does nothing else with the turn; the rest of the record is there when you want it.

## Core vocabulary: capability, harness, host, and workspace

Four more terms recur throughout Everruns. They describe how behavior reaches an agent and where a session's work happens. A **capability** bundles a tool definition with the two things that tool cannot work without: the system prompt addition that teaches the model when to call the tool, and the session state the tool needs at run time, such as filesystem mount points or secrets, and sometimes a dependency on another capability. A tool definition is a name with an argument schema and a handler. Attaching behavior to an agent as a capability keeps these concerns together instead of scattering them across your code.

The question "what environment am I running in?" is answered by the **harness**. It supplies model defaults and baseline tools, and it decides network access; the same harness is reused across many agents. The runtime layers the agent's own settings on top of the harness, then the session's settings on top of the agent's.

Underneath all of this sits the **host**, the layer that actually executes agents. It owns the immediate in-process driver, shared effect application, backend composition, event persistence, and low-level in-process execution. The Framework sits above the host, and your code stays there with it.

Where a session's work happens is its **workspace**, which is logical project lineage rather than an alias for a directory. Every session sees its workspace through the same portable `/workspace` path, regardless of whether the underlying storage is a Git worktree or a remote volume. Each stable, mutable view of that lineage is a workspace head, represented by the `WorkspaceHead` type and owned by a workspace provider.

## Typed tools from plain async functions

A tool is a function the model can choose to call. In Everruns you write it as an ordinary async Rust function and place one attribute above it.

````rust
/// Convert Celsius to Fahrenheit.
#[everruns::tool]
async fn fahrenheit(celsius: f64) -> f64 {
    celsius * 1.8 + 32.0
}
````

That is a complete tool. The doc comment becomes the tool's description and the typed argument becomes a JSON Schema for the tool's parameters. The attribute wraps the body in an `everruns::FunctionTool`, so the schema and the JSON wiring are generated rather than written by hand. Arguments and return values are ordinary Rust types. The return may be a plain value, as above, or a `Result` whose error type is `String`.

````rust
/// Add two integers.
#[everruns::tool]
async fn add(left: i64, right: i64) -> Result<i64, String> {
    Ok(left + right)
}
````

You add the tool to an agent by passing the function to the builder's `tool` method. Nothing else is required: not a trait implementation, a registration struct, a schema file, a capability registry, a tool record, or a backend store. The `hello` example agent does this with a `current_time` tool, and its instructions tell the model when to use it.

````rust
use everruns::{Agent, OpenAI};

/// Return the current Unix time in seconds.
#[everruns::tool]
async fn current_time() -> Result<String, String> {
    // ...
}

let agent = Agent::builder()
    .name("hello")
    .instructions("Use current_time to answer time questions. Be terse.")
    .provider(OpenAI::from_env()?)
    .model("gpt-5.6-terra")
    .tool(current_time())
    .build()?;
````

This example reaches a real model through `OpenAI::from_env()`, which needs the crate's `openai` feature enabled and reads the vendor's credentials from environment variables; chapter 04 names them. Once the tool is on the agent, dispatch is automatic. Your code sends a message and waits. The Framework runs the tool the model chose and feeds the result back to the model, and the returned `Turn` reports the calls that happened in `turn.tool_calls`. Complete example agents in the Everruns repository, such as `incident-commander-agent`, define every one of their tools this way, with `#[everruns::tool]` on an async function.

## Provider-neutral models and the offline default

A model names what to run; a provider is the vendor driver that runs it. `Model::simulated("Acknowledged.")` named a stand-in, while `.provider(OpenAI::from_env()?)` paired with `.model("gpt-5.6-terra")` named a real vendor's model, and the agent description had the same shape in both cases. That is deliberate. Every model vendor sits behind one uniform driver interface, so the same agent and prompt, with the same capabilities, run unchanged whether the model is served by OpenAI, Claude, Gemini, or any OpenAI-compatible endpoint. The provider boundary is open as well: a new provider arrives without a new closed enum variant, and your application code carries no provider-specific branch.

The crate's default feature set is declared in its manifest.

````toml
default = ["macros", "capabilities", "builtins", "filesystem"]
````

Those four features give you typed tools (`macros`), capabilities, built-ins, and the session filesystem. With that set the crate compiles and runs fully offline: no provider, shell, web, Lua, MCP, SQL, server, or worker integration is included, the dependency graph is free of LLM provider crates and HTTP clients, and the `providers` module is absent from the crate until you enable a provider feature such as `openai`. This is why the quickstart path is a deterministic agent: `Model::simulated` lets a unit test build an agent and run sessions with the network unplugged and no API key in the environment. When you are ready for a real model you enable the matching provider feature and change the model line; until then the only models available to a default build are simulated ones.

## Explicit capability boundaries

Filesystem, shell, web, Lua, and MCP integrations are each a named implementation boundary you choose to cross, and you cross it by toggling a Cargo feature on the `everruns` crate.

- `filesystem`: the session filesystem, on by default
- `bashkit`: a sandboxed shell
- `web-fetch` and `duckduckgo`: web access
- `lua`: Lua
- `mcp` and `mcp-stdio`: MCP invocation

Behind those flags, the core contract crate, `everruns-core`, defines the contracts for capabilities, tools, filesystems, egress, and MCP invocation, but it contains no interpreter, network, process, or filesystem code. Focused crates own that code, and the host links only the integrations your features select. Your trust boundaries are therefore visible in your manifest: a reader of `Cargo.toml` can tell whether this binary can run a shell or reach the network before reading a line of Rust.

## Events and cooperative cancellation

You can observe a complete agent turn as a typed event stream by asking the session for its events.

````rust
let mut events = session.events();
````

That one call installs an in-process subscriber. It does not expose the runtime's event buses or the core event types; you receive the typed events the session emits and nothing lower. The subscriber runs in your process alongside the engine. Cancellation is the other half: you can watch a turn as it runs and stop in-flight work cooperatively.

## One turn model, four surfaces: Framework, Runtime, SDKs, Platform

There is one turn model. A turn runs identically whether your application executes it in-process or the Everruns Platform executes it with durable checkpoints, because both paths converge on the same engine state machine in the `everruns-engine` crate, whose phases are named Input, Reason, and Act. Four surfaces sit on that shared model, and you choose among them by how you want to run agents.

- Framework: the application-facing `everruns` crate and its public library experience. Agents run in your process, and you own the process, the deployment, the integrations, and the data path.
- Advanced host crates, also called the Runtime: low-level execution-host composition through `everruns-host` and its focused sibling crates.
- SDKs: remote clients that call a running Everruns server. They do not embed Framework execution in the client process.
- Platform: the control plane, server, workers, UI, durable storage, and deployment topology used to operate agents as a service.

Normal library users start with the Framework, and the documented starting point is to embed Everruns in the Rust application you are building. Runtime is not a synonym for Framework; it means low-level host execution, and you choose it only when you need to compose a host yourself. You cross into custom backends only when you must replace storage or orchestration beneath the Engine, and moving from the embedded Framework to a distributed host, whether self-hosted or on the hosted Platform, is a deliberate and separate step.

*2026-09-17 03:24 - claude-fable-5.1*
