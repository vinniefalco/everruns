<!-- source: everruns/everruns @ 6cf2c15e5 (crate everruns 0.20.0). The working tree fast-forwarded to 7d9e07a0b (0.21.1) during extraction; identifiers and examples were audited against 6cf2c15e5. -->
# Your First Agent

An agent that answers a prompt takes one crate and about fourteen lines of Rust. The default build runs offline against a simulated model, so nothing here needs an API key until you decide to talk to a live provider.

## Adding `everruns` to a Cargo project

### One crate and a runtime

Two commands add everything the offline build needs.

````bash
cargo add everruns
cargo add tokio --features macros,rt-multi-thread
````

The first command adds the application-facing crate with no model provider compiled in. The second adds tokio, the async runtime the agent loop runs on. The `macros` feature supplies the `#[tokio::main]` attribute and `rt-multi-thread` supplies the scheduler; the Framework requires no other tokio features. The whole loop executes inside your own async main, in one async context with nothing else running.

Here is the smallest complete program. It describes an agent and runs one turn against it, then prints the result.

````rust
use everruns::{Agent, Engine, Model};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let agent = Agent::builder()
        .instructions("You are a helpful assistant.")
        .model(Model::simulated("4"))
        .build()?;

    let turn = Engine::new().create(agent).send_and_wait("What is 2 + 2?").await?;
    println!("{}", turn.response);
    Ok(())
}
````

Run it with `cargo run` and it prints `4`, the reply scripted into the simulated model. The program imports only `everruns`. Lower-level core and host crates exist for advanced execution hosts, and an ordinary application never depends on them. The entry point returns a boxed error, so every Framework error propagates with `?`. The shipped standalone examples return the `Send + Sync` form, `Box<dyn std::error::Error + Send + Sync>`, and Framework errors convert into either.

To talk to a real model instead, turn on the OpenAI provider when you add the crate and put a key in the environment.

````bash
cargo add everruns --features openai
export OPENAI_API_KEY=sk-...
````

Without that key in the environment, the live program fails at start.

### The manifest and the imports

The commands above produce a manifest like this one. A real application depends on a published version. The standalone examples in the framework repository point at the crate source with a `path` dependency instead, and a project that vendors or submodules the framework needs an empty `[workspace]` table to stay out of the parent workspace; the rest of the manifest is the same either way.

````toml
[package]
name = "first-agent"
edition = "2024"

[dependencies]
everruns = "0.20.0"
tokio = { version = "1.50", features = ["macros", "rt-multi-thread"] }
````

Target edition 2024 and a recent stable toolchain. The published crate is `0.20.0` and follows its own semantic version rather than a workspace-wide product version, and its maintainers commit to preserving its public application contract. Some documentation still shows `0.17` pins for the core and host crates. If you ever depend on more than one everruns crate, keep them on one matching version line. The project documentation describes it as under active development, so expect the API to move.

You can name the types you use at the crate root, or bring in the whole common path with the prelude.

````rust
use everruns::{Agent, Engine, Model};
````

````rust
use everruns::prelude::*;
````

The prelude resolves everything needed to describe an agent and run turns. Both styles appear in the shipped examples and in this chapter. Beyond providers, tools, sessions, and turns, the crate also covers live events, cancellation, background work, files, workspaces, lifecycle hooks, MCP, and durable local state.

The API reference is at `https://docs.rs/everruns`. The framework quickstart and overview are at `https://docs.everruns.com/framework/`, separate from the platform documentation, and the quickstart is the same install-and-run-offline sequence this chapter follows.

## Feature flags and what each pulls in

A dependency line with no `features` key gives you the default set.

````toml
everruns = "0.20.0"
````

That line activates `default = ["macros", "capabilities", "builtins", "filesystem"]` and nothing else. The default build is offline and compiles no provider. Each default feature does one job.

- `macros` supplies the `#[everruns::tool]` attribute. It pulls in the `everruns-macros` crate along with `schemars` for schema generation and `serde` derive support.
- `capabilities` enables authoring custom typed capabilities. A capability is a named bundle of tools and the behavior that goes with them.
- `builtins` enables the built-in capability set.
- `filesystem` enables the contained per-session filesystem.

The opt-in features layer integrations on top. `openai` and `local` are the two that matter for this chapter.

- `openai` turns on the OpenAI model provider.
- `local` turns on durable local persistence, the alternative to the in-memory default.
- `bashkit` turns on the sandboxed shell integration.
- `web-fetch` turns on web fetch.
- `mcp` turns on MCP server integration.

You can combine integrations in one install command.

````bash
cargo add everruns --features openai,bashkit,web-fetch,mcp
````

Features are additive. A manifest that lists the shell and the provider keeps all four defaults active and layers those two on top.

````toml
everruns = { version = "0.20.0", features = ["bashkit", "openai"] }
````

Listing only `openai` adds only the provider, and omitting the `features` key still yields all four defaults. The only way to remove a default is to disable default features explicitly.

When a feature is on, the prelude includes that feature's types automatically, so there is no separate import path to learn per integration. Turn on `openai` and `use everruns::prelude::*;` resolves `OpenAI`; turn on `bashkit` and the same import resolves the shell type. One further feature name, `jsonl`, is retained from the `0.17` line as an empty no-op so older feature lists keep compiling; it enables nothing.

## Describing an agent with `Agent::builder()`

### From a simulated model to a live one

An agent description begins at `Agent::builder()` and ends at `build()`. The two calls between them are the only required ones.

````rust
use everruns::{Agent, Model};

let agent = Agent::builder()
    .instructions("You are a helpful assistant.")
    .model(Model::simulated("4"))
    .build()?;
````

The system prompt goes in `instructions`, which is required: blank or whitespace-only text is rejected at build time with the message `agent instructions must not be blank`. By convention it is the first call after opening the builder. The second required call is `model`, which selects what runs the reasoning; leaving it out is a build error. The simulated model is the default path for the framework's own tests and documentation because it needs no provider and no network, which is also why the default features stay offline. Here it is `Model::simulated("4")`, returning the string `"4"` as the agent's response to any input.

A live agent adds a name, a provider, a model id, and a tool. Define the tool before the chain. A tool is an ordinary async Rust function with the `#[everruns::tool]` attribute; the attribute derives the tool's JSON Schema and adapter from the function's signature and uses the doc comment as the description the model sees.

````rust
use everruns::{Agent, OpenAI};

/// Return the current Unix time in seconds.
#[everruns::tool]
async fn current_time() -> Result<u64, String> {
    std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .map(|elapsed| elapsed.as_secs())
        .map_err(|err| err.to_string())
}

let agent = Agent::builder()
    .name("assistant")
    .instructions("Answer concisely.")
    .provider(OpenAI::from_env()?)
    .model("gpt-5.6-terra")
    .tool(current_time())
    .max_iterations(12)
    .build()?;
````

Two of the new lines, `provider(OpenAI::from_env()?)` and `model("gpt-5.6-terra")`, are the entire difference between simulated and live. The first attaches a provider built from `OPENAI_API_KEY`; the second passes a plain model id string that the provider recognizes. The rest of the program does not change. An agent accepts exactly one provider, and a missing provider on a live model is a build error.

The name is optional; `name("assistant")` overrides the default `"agent"`, and the value is a stable identifier that shows up in sessions as well as in events and logs. Registering the tool is `tool(current_time())`: the attribute generated a zero-argument constructor with the same name as the function, and calling it yields the tool value the builder accepts. Tool names must be valid and unique on an agent.

The last call before `build()`, `max_iterations(12)`, caps how many reason-and-act iterations one run may perform. It is a hard bound, so when the cap is reached the run stops. The shipped examples use 6 and 12, and the cap keeps a model from searching and reading indefinitely so cost stays predictable. More builder calls exist for capabilities, files, workspaces, and parallel tool calls.

### What `build()` checks

Validation happens in one place: `build()` checks the whole description and returns `Result<Agent, BuildError>`. With `?` in a main that returns a boxed error, a misconfigured agent fails at construction, before any engine or session exists. Nothing is validated lazily at run time.

````rust
use everruns::{Agent, BuildError, Model};

let err = Agent::builder()
    .instructions("   ")
    .model(Model::simulated("Sure."))
    .build()
    .unwrap_err();

assert_eq!(err, BuildError::BlankInstructions);
````

Eleven failure conditions are documented. The ones you can hit with this chapter's material are `BlankInstructions`, `MissingModel`, `InvalidToolName`, `InvalidToolSchema`, `DuplicateTool`, `MissingProvider`, and `MultipleProviders`; the remaining four belong to capabilities and to the MCP and workspace integrations. `MultipleProviders` prints `agent accepts one provider; registered providers: [...]` with the list of what was registered. Every variant compares by equality and implements the standard error trait with a readable message. The enum is marked non-exhaustive so it can grow, and no backend error leaks through it.

The builder itself is `Clone`, so a base configured once can be cloned and finished differently to derive variant agents from one shared description.

````rust
let base = Agent::builder()
    .instructions("You are a helpful assistant.");

let terse = base.clone().model(Model::simulated("Yes.")).build()?;
let verbose = base.model(Model::simulated("Yes, and here is why.")).build()?;
````

Debug output on both `Agent` and `AgentBuilder` lists capability ids and workspace information without exposing runtime internals.

### Organizing a real agent

Instructions do not have to be an inline literal. The shipped standalone examples keep the prompt in a markdown file next to the source and embed it at compile time, so the prompt is editable as plain text with no runtime file I/O and no change to the builder.

````rust
let agent = Agent::builder()
    .instructions(include_str!("resources/instructions.md"))
    .model(Model::simulated("Acknowledged."))
    .build()?;
````

Those examples also wrap construction in a factory function that takes credentials as an argument. One definition is then built from `main` and from tests, and the key is injected by the caller rather than read inside the builder.

````rust
// src/agent.rs
use everruns::{Agent, BuildError, OpenAI};

pub fn build(api_key: String) -> Result<Agent, BuildError> {
    Agent::builder()
        .instructions(include_str!("resources/instructions.md"))
        .provider(OpenAI::new(api_key))
        .model("gpt-5.6-terra")
        .max_iterations(12)
        .build()
}
````

````rust
// src/main.rs
mod agent;
mod tools;

let api_key = std::env::var("OPENAI_API_KEY")?;
let agent = agent::build(api_key)?;
````

With that split, `main.rs` handles input and host setup while `agent.rs` owns the builder; the `#[everruns::tool]` definitions go in `tools.rs` and are attached inside `agent::build`, not in `main`.

## Creating an Engine and a session

A built `Agent` does nothing on its own. You hand it to an `Engine` that your application owns.

````rust
use everruns::{Agent, Engine, Model};

let agent = Agent::builder()
    .instructions("Remember the conversation.")
    .model(Model::simulated("Acknowledged."))
    .build()?;

let engine = Engine::new();
let session = engine.create(agent);
````

Constructing the engine takes no arguments: `Engine::new()` has no builder and no config struct, and `Engine::default()` does the same thing. The engine is a concrete application object rather than a trait you implement, and it is the normal Framework entry point. It holds agent snapshots, sessions, history, and resume authority; its `Debug` output renders as `Engine { sessions: <count> }`.

Creating the session is synchronous. `engine.create(agent)` generates a fresh session id and records the agent, then returns a live `Session` bound to that engine, so no `.await` is needed until you send something. The signature is `pub fn create(&self, agent: Agent) -> Session`, and because it takes the agent by value, creating a session consumes it.

Cloning an agent is cheap, so clone before you create when you intend to reuse it.

````rust
let engine = Engine::new();
let support = engine.create(agent.clone());
let billing = engine.create(agent);
````

The intended shape is to describe an agent once, then create independent multi-turn conversations from it, in one engine or across many. Each session has its own history, so what you send to `support` is invisible to `billing`.

Everything in this chapter runs in memory, and session state lasts only as long as the engine, so keep the engine alive as long as you need its sessions.

An older name for the engine remains as a compatibility alias. `InMemoryEngine` is declared as `pub type InMemoryEngine = Engine;`, the same type rather than a second implementation, so mixing the two names in one program is harmless. New code uses `Engine`.

## Sending a turn and reading `turn.response`

### Request and response

Sending a message is one call: `send_and_wait` blocks until the turn it starts has finished.

````rust
let session = Engine::new().create(agent);

let turn = session.send_and_wait("What time is it?").await?;
println!("{}", turn.response);
````

The call takes the user's message text directly and returns a `Turn`. `turn.response` is a plain string holding the concatenated text output of the turn. The signature is `pub async fn send_and_wait(&self, input: impl Into<InputMessage>) -> Result<Turn, RunError>`, and `?` propagates a `RunError` the same way it propagated a `BuildError` earlier. It blocks through any tool round trip. The Framework documents it as the request/response convenience for when you do not need a live timeline of what happened inside the turn; a streaming form exists for when you do.

Call it repeatedly on the same session to hold a conversation. Later sends see earlier turns.

````rust
let first = session.send_and_wait("My name is Ada.").await?;
let second = session.send_and_wait("What is my name?").await?;
println!("{}", second.response);
````

When you want a single turn and nothing else, the shortest path from agent to answer is `run`, which collapses everything from engine construction to the awaited turn into a single expression.

````rust
let turn = Engine::new()
    .create(agent)
    .run("Confirm the agent is configured.")
    .await?;
````

Like `send_and_wait`, `run` takes the message text and returns a turn whose `response` is the concatenated output.

The `Turn` is a structured result. Alongside `response` it has a `success` flag and an `error` field, which is what a test checks.

````rust
use everruns::{Agent, Engine, Model};

#[tokio::test]
async fn simulated_agent_answers() -> Result<(), Box<dyn std::error::Error>> {
    let agent = Agent::builder()
        .instructions("You are a helpful assistant.")
        .model(Model::simulated("4"))
        .build()?;

    let result = Engine::new()
        .create(agent)
        .send_and_wait("What is 2 + 2?")
        .await?;

    assert!(result.success, "turn should succeed: {:?}", result.error);
    assert_eq!(result.response, "4");
    Ok(())
}
````

In the assertions, `success` is a `bool` and `error` is `Debug`-printable, while `response` compares directly against a `&str`. The turn record has more fields than these; the ones here are enough to print an answer or assert on one.

### Letting the model call a tool

Tool invocation is automatic: once a tool is registered on the builder, a plain `send_and_wait` is all the caller does. Here is the live chain from the builder section, completed into a program.

````rust
use everruns::{Agent, Engine, OpenAI};

/// Return the current Unix time in seconds.
#[everruns::tool]
async fn current_time() -> Result<u64, String> {
    std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .map(|elapsed| elapsed.as_secs())
        .map_err(|err| err.to_string())
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let agent = Agent::builder()
        .name("assistant")
        .instructions("Answer concisely.")
        .provider(OpenAI::from_env()?)
        .model("gpt-5.6-terra")
        .tool(current_time())
        .build()?;

    let session = Engine::new().create(agent);
    let turn = session.send_and_wait("What time is it?").await?;
    println!("{}", turn.response);
    Ok(())
}
````

Build it with the `openai` feature enabled and `OPENAI_API_KEY` set. During the turn the model may decide to call `current_time`. The Framework runs the function and hands the result back to the model, and the model's final response arrives in `turn.response` as before. `main` contains no dispatch loop and no match on tool names; `send_and_wait` is the only Framework call it makes after `build()`.

## Running the shipped examples

Most of the examples that ship with the crate run with one command.

````bash
cargo run -p everruns --example engine_sessions
````

That is the general form, `cargo run -p everruns --example <name>`, and for an offline example it is the whole command. `engine_sessions` runs offline with the simulated model and no flags; it demonstrates owning a concrete engine and running isolated sessions inside it, then resuming sessions scoped to that engine. The example catalog in the crate's `examples/README.md` lists the exact command for every example.

Model-backed examples add the provider feature and need the key in the environment.

````bash
export OPENAI_API_KEY=sk-...
cargo run -p everruns --features openai --example hello
````

Without the feature the example does not compile, because it is declared with `required-features = ["openai"]`. Without the key it fails at start. Most live examples use the model id `gpt-5.6-terra`.

The smallest useful Everruns program is `hello`: 56 lines including imports and printing, in which an agent is composed and a typed tool is run, then the turn is inspected and the event stream observed. `production_agent`, run the same way, shows a production-shaped setup with tools, files, a tool safety boundary, and multi-turn use.

The examples sort by what they need.

- No flag: `capability_configuration`, `canonical_events`, `session_work`, `workspace_policy`, `engine_sessions`. All run offline with the simulated model.
- `--features local`: `session_history` and `workspace_heads`. These also run offline, and `workspace_heads` works against a local Git repository.
- `--features openai` plus `OPENAI_API_KEY`: `hello`, `production_agent`, `github_monitor`, `subagents`, `observe_and_cancel`, `advanced_capability`, `lifecycle_hooks`.

## The offline default and `--no-default-features`

Depending on `everruns` with its defaults gives you an agent host that stays offline.

````toml
[dependencies]
everruns = "0.20.0"
````

With that line, no provider, shell, web, Lua, MCP, SQLx, server, or worker integration is activated, and building or running requires no database or network connection, and no provider credential. The smallest build is deterministic and credential-free, and it is independent of the hosted platform. Optional integrations stay feature-gated so they never enlarge the default build.

The standard build pulls in no HTTP or TLS stack at all, because the provider and agent-to-agent integrations that would need one are off. Enabling `openai` is what brings a network client into the dependency graph, and the `a2a` opt-in brings a second one, the `a2a-client` crate with its own HTTP stack.

Conversely, disabling default features reduces the crate to a bare re-export facade that still compiles. From there you add back only what you need.

````toml
[dependencies]
everruns = { version = "0.20.0", default-features = false, features = ["macros"] }
````

This build has the tool attribute and none of the other three defaults. `macros` and `capabilities` are gated independently of each other, so you can take either without the other. Applications that need only the open Framework contracts start from this line and list their integrations explicitly.

*2026-09-17 03:24 - claude-fable-5.1*
