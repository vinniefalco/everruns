<!-- source: everruns/everruns @ 6cf2c15e5 (crate everruns 0.20.0). The working tree fast-forwarded to 7d9e07a0b (0.21.1) during extraction; identifiers and examples were audited against 6cf2c15e5. -->
# Tools and the Macro

An agent without tools can only talk. Give it tools and it can look up an order or run your regression suite, and the Framework handles every step between the model's decision to call a tool and the result it reads back. In `everruns` the unit of that work is one attribute on one async Rust function. `#[everruns::tool]` turns the function's signature into a JSON Schema, its doc comment into the description the model sees, and the function itself into a value you pass to `.tool(...)` on the agent builder. You write no schema by hand, no registration struct, no trait implementation, and no dispatch loop. From that one attribute the chapter works outward to the core tool types beneath it and then to the capability contract that packages several tools with their own instructions into one reusable unit. Every code sample in this chapter runs offline against `Model::simulated`, and the macro path needs no dependency beyond `everruns` and `tokio`.

## The `#[everruns::tool]` macro and its attribute options

### The minimal tool

Here is a complete tool and the agent that uses it.

````rust
use everruns::{Agent, Engine, Model};

/// Look up an order in the read-only public order namespace.
#[everruns::tool]
async fn lookup_order(order_id: String) -> Result<String, String> {
    if !order_id.starts_with('A') {
        return Err("order_id must use the public A-NNN format".into());
    }
    Ok(format!("Order {order_id}: shipped"))
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let agent = Agent::builder()
        .instructions("Use the lookup tool to answer order questions.")
        .model(Model::simulated("Order A-100 has shipped."))
        .tool(lookup_order())
        .build()?;

    let turn = Engine::new().create(agent).send_and_wait("Where is order A-100?").await?;
    println!("{}", turn.response);
    Ok(())
}
````

The function body is ordinary Rust. The attribute reads the signature, derives a JSON Schema with one required string property named `order_id`, and generates the adapter that deserializes the model's arguments into that parameter and serializes the return value back. The `///` doc comment is the description the model reads when deciding whether to call the tool. Nothing else in the file is Framework-specific.

The attribute also replaces `lookup_order` with a zero-argument constructor of the same name. The call `lookup_order()` inside `.tool(lookup_order())` does not run the body; it returns a tool value, and the builder registers it. When the model asks for `lookup_order` during a turn, the Framework runs the body with the model's arguments and feeds the result back, so the program above never dispatches anything by hand.

The macro is a procedural macro implemented in a separate macro crate and re-exported from `everruns` behind the default-enabled `macros` feature. Use the re-export and never depend on the macro crate directly. The generated code resolves `serde`, `schemars`, and `serde_json` through the framework, so your manifest lists `everruns` and `tokio`, plus a provider crate if you use one.

### Description, name, and visibility

The doc comment may sit above the attribute or below it. The macro trims each doc line and joins the lines with newlines, then trims the result, so a multi-line comment becomes a multi-line description. There is no separate description call on the builder. When you want the description in the attribute instead, or you want the model to see a different name than the Rust function has, use the two attribute options.

````rust
/// This comment is ignored because the option below wins.
#[everruns::tool(name = "add", description = "Add two numbers.")]
async fn adder(a: i64, b: i64) -> i64 {
    a + b
}
````

The model sees a tool called `add` described as "Add two numbers." The explicit `description` option wins whenever a doc comment is also present. The Rust name is unchanged, so you still register this tool with `.tool(adder())`. Both options take string literals, and `name` and `description` are the only options the attribute accepts.

The generated constructor keeps the visibility of the original function, which is what lets you define tools in their own module.

````rust
// src/tools.rs

/// Return the source of one exposed file.
#[everruns::tool]
pub async fn inspect_change(path: String) -> Result<String, String> {
    source(&path).map(str::to_owned)
}

fn source(path: &str) -> Result<&'static str, String> {
    match path {
        "contract.md" => Ok("# Contract\n..."),
        _ => Err("Only sample_payment.rs, contract.md, and regression.rs are exposed.".into()),
    }
}
````

````rust
// src/main.rs
mod tools;

let agent = Agent::builder()
    .instructions("Review the change.")
    .model(Model::simulated("Reviewed."))
    .tool(tools::inspect_change())
    .build()?;
````

A `pub async fn` yields a `pub fn` constructor that `main.rs` reaches as `tools::inspect_change()`. The private helper `source` stays private and is never exposed as a tool, because only the annotated function gets a constructor. This split, with agent construction in one file and tools in another, is the layout the shipped multi-file examples use.

## Accepted signatures, argument types, and return shapes

### Parameters

A required argument is a plain typed parameter. An optional argument is `Option<T>`. The Rust parameter names become the argument names the model uses, and the parameter types produce the JSON Schema, so there is no separate arguments struct to declare.

````rust
/// Report the weather for a city. Units default to celsius.
#[everruns::tool]
async fn weather(city: String, units: Option<String>) -> Result<String, String> {
    let units = units.unwrap_or_else(|| "celsius".to_string());
    Ok(format!("{city}: 21 degrees {units}"))
}
````

The schema for `weather` marks `city` as required and `units` as optional. The model may omit `units`, and the body receives `None`. An empty parameter list is also allowed, which suits tools that read state rather than take input. Any argument type that implements `Deserialize` and `JsonSchema` works as a parameter, including your own structs, so a tool can accept one structured record instead of a flat list of fields.

To change the key the model supplies without renaming the Rust parameter, put `#[tool(rename = "...")]` on that parameter.

````rust
/// Add two integers.
#[everruns::tool]
async fn add(#[tool(rename = "left")] a: i64, b: i64) -> i64 {
    a + b
}
````

The schema and the accepted JSON use `left`; the body keeps using `a`. The `rename` key is the only parameter-level option.

### Return shapes

Return `Result<T, E>` where `E` implements `Display` and an `Err` value becomes a model-visible tool error. The run continues and the model reads the message, so it can correct its call. This is the shape for validation: check inside the body and return `Err("...")` rather than panicking.

````rust
use std::time::{SystemTime, UNIX_EPOCH};

/// Return the number of seconds since the Unix epoch.
#[everruns::tool]
async fn current_time() -> Result<String, String> {
    let seconds = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .map_err(|error| error.to_string())?
        .as_secs();
    Ok(format!("{seconds} seconds since the Unix epoch"))
}
````

Return a plain serializable `T`, including `()`, and it is serialized to JSON as the success response. A returned `String` is the text the model reads. Return values must implement `Serialize`. The macro detects a fallible return by the final path segment of the return type, so aliases such as `anyhow::Result<T>` and `std::io::Result<T>` are treated as `Result` returns and get the same `Err` handling.

Two failure paths look alike and behave differently. If the model sends invalid or mistyped arguments, the model receives an error that names the tool and the bad field, in the form `invalid arguments for tool `weather`: ...`, and it can fix its own call; this is not a host failure. If your return value fails to serialize, that is a host-side error of the form `failed to serialize result of tool `weather`: ...`, and the model does not see it.

You register several tools with different shapes by chaining `.tool(...)` on one builder.

````rust
let agent = Agent::builder()
    .instructions("Use the tools.")
    .model(Model::simulated("ok"))
    .tool(weather())
    .tool(add())
    .tool(current_time())
    .build()?;
````

## Compile-time rejections from the trybuild cases

The macro validates the whole signature before it expands anything, so a rejected function produces one top-level error at the offending token and no cascade of errors from generated code. Each rule below is pinned by a compile-fail fixture in the `everruns` test suite, and a single pass fixture proves the supported signatures compile without a hand-written schema or trait implementation. The snippets show the smallest function that trips each rule, followed by the exact diagnostic.

### Signature rules

The function must be `async fn`. The Framework awaits every tool invocation.

````rust
use everruns::tool;

/// Echo a number.
#[tool]
fn not_async(x: i64) -> i64 {
    x
}
````

````text
error: `#[everruns::tool]` requires an `async fn`
````

The carets point at `fn`, and the fix is the one word `async`. The diagnostic always reads `#[everruns::tool]` even when you imported the macro and applied it as `#[tool]`.

The function must not take a `self` receiver. The runtime has no instance to supply, so a method inside an `impl` block cannot use `self`, `&self`, or `&mut self`.

````rust
struct Calculator;

impl Calculator {
    /// Add to the accumulator.
    #[everruns::tool]
    async fn add(&self, x: i64) -> i64 {
        x
    }
}
````

````text
error: `#[everruns::tool]` does not support a `self` receiver
````

Every parameter must be a plain named identifier of the form `name: Type`, because every argument needs a stable name in the schema. A tuple-destructuring pattern is rejected; write `a: i64, b: i64` instead.

````rust
/// Add a pair.
#[everruns::tool]
async fn add_pair((a, b): (i64, i64)) -> i64 {
    a + b
}
````

````text
error: `#[everruns::tool]` parameters must be plain identifiers (`name: Type`)
````

The function must have no generic parameters and no `where` clause. The schema is generated from concrete argument types, and an unresolved `T` has no schema to emit.

````rust
/// Return the input.
#[everruns::tool]
async fn identity<T>(value: T) -> T {
    value
}
````

````text
error: `#[everruns::tool]` does not support generic parameters
````

The caret sits under the `T` in `<T>`. A `where` clause fails with its own message.

````text
error: `#[everruns::tool]` does not support `where` clauses
````

### Attribute rules

A tool must have a description. With neither a `///` doc comment nor a `description` option, compilation fails and the message names both remedies.

````rust
#[everruns::tool]
async fn ping() -> String {
    "pong".into()
}
````

````text
error: `#[everruns::tool]` requires a description: add a doc comment or `#[everruns::tool(description = "â€¦")]`
````

The carets sit under the function name.

The attribute accepts exactly two options, `name` and `description`. Any other key fails with a message that lists them.

````rust
/// Reply with pong.
#[everruns::tool(title = "Ping")]
async fn ping() -> String {
    "pong".into()
}
````

````text
error: unknown `everruns::tool` option `title`; expected `name` or `description`
````

Option values must be string literals.

````rust
/// Reply with pong.
#[everruns::tool(name = ping_name)]
async fn ping() -> String {
    "pong".into()
}
````

````text
error: expected a string literal
````

A parameter-level `#[tool(...)]` attribute accepts only `rename`.

````rust
/// Echo a value.
#[everruns::tool]
async fn echo(#[tool(alias = "v")] value: String) -> String {
    value
}
````

````text
error: expected `rename = "â€¦"`
````

## `FunctionTool` and `ToolResponse` for runtime-defined schemas

### Building and registering a function tool

The macro needs the schema at compile time. When the schema is only known at run time, because it comes from a database row or a plugin catalog, construct the tool directly with `FunctionTool::new(name, description, json_schema, handler)`. `FunctionTool` and `ToolResponse` are exported from `everruns` with no feature gate.

````rust
use everruns::{Agent, FunctionTool, Model, ToolResponse};
use serde_json::{json, Value};

fn weather_tool() -> FunctionTool {
    FunctionTool::new(
        "get_weather",
        "Report the sky condition for a city.",
        json!({
            "type": "object",
            "properties": { "city": { "type": "string" } },
            "required": ["city"],
            "additionalProperties": false
        }),
        |arguments: Value| async move {
            let city = arguments["city"].as_str().unwrap_or("");
            if city.is_empty() {
                return Ok::<_, String>(ToolResponse::error("city must not be empty"));
            }
            Ok(ToolResponse::json(json!({ "city": city, "sky": "clear" })))
        },
    )
}

let agent = Agent::builder()
    .instructions("Answer weather questions with the tool.")
    .model(Model::simulated("Clear skies."))
    .tool(weather_tool())
    .build()?;
````

The schema is a JSON value, typically an object with `properties` and `required`. The handler is an async closure that receives the model's arguments as a JSON value and returns a `Result`. You register the value itself with `.tool(...)`; there is no constructor call because you built the value yourself. The schema and arguments are `serde_json::Value` values, so this path names `serde_json` in your own code; the shipped docs add `serde_json` to the manifest as above, and the re-export at `everruns::capability::serde_json` works equally well. Function tools and string capability references can be mixed on the same builder.

### Responses, errors, and build-time validation

The handler has four ways to answer. `ToolResponse::json(value)` and `ToolResponse::text(text)` are explicit structured successes. A bare JSON value or a `String` returned inside `Ok` is also a success and is passed to the model unchanged. `Ok(ToolResponse::error(message))` is a model-visible tool error: the turn continues and the model reads the message. In the example above, an empty city produces that error.

An `Err` returned from the handler is instead treated as an internal error and redacted before it reaches the model, which sees only "An internal error occurred while executing the tool". The full error is logged with the tool name and call id. The turn still succeeds and nothing panics. Use this channel for failures the model did not cause. The macro never uses it: a macro tool's own `Err` is mapped to a model-visible error first. So when the model should read the message, return `Ok(ToolResponse::error(...))` from a function tool, or `Err` from a macro tool.

The builder validates every function tool before an agent is produced, and the errors are typed `BuildError` variants you can assert on.

````rust
use everruns::BuildError;

let make = || {
    FunctionTool::new(
        "lookup",
        "Look up a record.",
        json!({ "type": "object", "properties": {} }),
        |_: Value| async move { Ok::<_, String>(json!({})) },
    )
};

let err = Agent::builder()
    .instructions("Use the tools.")
    .model(Model::simulated("ok"))
    .tool(make())
    .tool(make())
    .build()
    .expect_err("duplicate names must fail");

assert_eq!(err, BuildError::DuplicateTool { name: "lookup".to_string() });
````

A tool name must be 1 to 64 characters, start with a letter or underscore, and contain only letters, digits, `_`, or `-`; names such as `2fast`, `has space`, and `dot.name` fail with `BuildError::InvalidToolName` carrying a reason such as `tool name must start with a letter or underscore (got '2')`. A tool's JSON Schema must be a JSON object, and if it declares a top-level `type` that value must be the string `"object"`; a schema with no `type` is accepted, and `{"type": "array"}` fails with `BuildError::InvalidToolSchema`. Two tools with the same name on one agent fail with `BuildError::DuplicateTool` naming the tool.

## Provider-facing tool types and the core `Tool` trait

The macro and `FunctionTool` both produce lower-level values defined in the core and provider crates beneath `everruns`. You work with those values directly when you implement the core `Tool` trait yourself or when you read tool calls and results off the event stream.

### The wire types

A tool definition the model receives is a builtin tool with a name, a description, and JSON Schema parameters. On the wire it is tagged `"type": "builtin"`. A second variant, tagged `"type": "client_side"`, describes a tool the client executes: the run pauses and waits for the client to submit a result, and its policy is always `client_side`.

Every tool has a policy. `auto` is the default and executes immediately; `requires_approval` gates the call behind human approval; `client_side` is the client-executed case above. An omitted policy means `auto`.

A tool can also declare semantic hints following the MCP annotation convention: `readonly`, `destructive`, `idempotent`, and `open_world`. Each is an `Option<bool>`, and unset means unknown. Hints are informational and enforce no policy. The hint set is built fluently with setters such as `with_readonly`, `with_destructive`, `with_idempotent`, `with_open_world`, and `with_long_running`, where `long_running` signals work over roughly five seconds so clients can show progress. Further hints include `requires_secrets`, a narration noun, capability attribution, and an opaque metadata blob. Metadata is persisted and shown to clients, so it must never contain credentials.

A model-issued invocation is a tool call with an `id`, a tool `name`, and JSON `arguments`. The outcome is a tool result correlated to the originating call id, with either success data or an error message, and optionally a list of images. An image is base64 content plus a media type such as `image/png`, and it reaches the model as a native image content block rather than as stringified JSON.

````json
{"id": "call_123", "name": "get_weather", "arguments": {"city": "Kyiv"}}
{"tool_call_id": "call_123", "result": {"temperature": 72}, "error": null}
````

Oversized results are truncated at a 64 KiB backstop. The text ends with a suffix explaining the truncation and suggesting quiet flags, pipes, or redirecting to a file. The primary limit is enforced earlier by a hook in the tool pipeline; the backstop applies only to output that hook did not already cut.

### Implementing `Tool`

You define a tool at the core level by implementing the `Tool` trait: a name, a description, a JSON Schema for parameters, and an async `execute` body that receives JSON arguments and returns a tool execution result. Every other method has a default. The identifiers below are exported by the core crate; `async_trait` comes from the `async-trait` crate, which the core trait itself uses.

````rust
use async_trait::async_trait;
use serde_json::{json, Value};

struct GetCurrentTime;

#[async_trait]
impl Tool for GetCurrentTime {
    fn name(&self) -> &str {
        "get_current_time"
    }

    fn description(&self) -> &str {
        "Return the current time."
    }

    fn parameters_schema(&self) -> Value {
        json!({ "type": "object", "properties": {} })
    }

    async fn execute(&self, _arguments: Value) -> ToolExecutionResult {
        ToolExecutionResult::success(json!({ "now": "12:00" }))
    }

    fn display_name(&self) -> Option<&str> {
        Some("Get Current Time")
    }

    fn policy(&self) -> ToolPolicy {
        ToolPolicy::RequiresApproval
    }

    fn hints(&self) -> ToolHints {
        ToolHints::default().with_readonly(true).with_idempotent(true)
    }
}
````

The three overrides at the bottom are optional. `display_name` gives clients a human-readable label separate from the technical name, which is the fallback. `policy` defaults to `Auto`; returning `RequiresApproval` routes the call through the approval flow. `hints` defaults to all unknown. Calling `to_definition()` on any tool converts it into the provider-ready definition bundling name, display name, description, parameters, policy, deferral, and hints.

The execution result has one constructor per outcome. `success(value)` forwards a JSON value to the model, and `success_with_images(value, images)` adds images; an empty list is treated as no images. `tool_error(message)` reports an expected failure such as a validation error or a missing resource, and the model receives it as `{"error": "..."}` with the message verbatim. `internal_error(error)` and `internal_error_msg(message)` are the core form of the redacted internal channel that a function tool reaches by returning `Err`. `connection_required(provider)` names a provider such as `brave_search`; the workflow pauses and the user is prompted to configure the connection.

## `ToolContext` and `ToolRegistry`

These are core-level types as well. You need the context when a core `Tool` needs to know which session is calling it, and the registry when you assemble tools for a host rather than for an agent builder.

### Reading the context from a tool

A core tool can receive a typed tool context during execution. To use it, override `execute_with_context`, whose default delegates to plain `execute`, and return `true` from `requires_context`.

````rust
use async_trait::async_trait;
use serde_json::{json, Value};

struct ScopedLookup;

#[async_trait]
impl Tool for ScopedLookup {
    fn name(&self) -> &str {
        "scoped_lookup"
    }

    fn description(&self) -> &str {
        "Look up records that belong to the calling session."
    }

    fn parameters_schema(&self) -> Value {
        json!({ "type": "object", "properties": { "query": { "type": "string" } }, "required": ["query"] })
    }

    async fn execute(&self, _arguments: Value) -> ToolExecutionResult {
        ToolExecutionResult::tool_error("scoped_lookup needs session context")
    }

    fn requires_context(&self) -> bool {
        true
    }

    async fn execute_with_context(&self, arguments: Value, context: &ToolContext) -> ToolExecutionResult {
        context.emit_progress(self.name(), "searching session records").await;
        let session = context.session_id;
        let records = records_for(session, arguments["query"].as_str().unwrap_or("")).await;
        ToolExecutionResult::success(json!({ "records": records }))
    }
}
````

The context holds `session_id`, the id of the session executing the tool, so the body can scope its work to the caller. It may include an optional session filesystem in `file_store`; when present, the tool reads and writes session files through it. `emit_progress(tool_name, message)` streams a `tool.progress` event on a best-effort basis. The context also holds a handle for adjusting the model's reasoning effort mid-turn.

The context may hold a cancellation token in `cancellation`, with the call-scoped semantics that the handler `Context` section at the end of this chapter spells out. `context.is_cancelled()` reports the current state, and a context with no token never reports cancellation.

A core tool can declare which runtime services it requires by overriding `required_context_services`. If a required service is absent, the tool is rejected at configuration time with an error of the form `tool "scoped_lookup" requires unavailable ToolContext service ...`, before it is ever advertised to the model. For tests or embedded use, `ToolContext::new(session_id)` builds a minimal context from a session id alone with every optional service absent.

### Assembling a registry

A registry holds core tools keyed by name and produces the model-facing definitions for all of them in one call.

````rust
let registry = ToolRegistry::builder()
    .tool(GetCurrentTime)
    .tool(ScopedLookup)
    .build();

let definitions = registry.tool_definitions();
````

The same registry can be built imperatively. `ToolRegistry::new()` creates an empty one, and `register`, `register_boxed`, and `register_arc` add tools by value, boxed, or Arc-wrapped; the builder mirrors them as `.tool(...)`, `.tool_boxed(...)`, and `.tool_arc(...)`. Registering a tool under a name that already exists replaces the earlier tool, and a registry can be inspected by name, have single tools unregistered, or be emptied. `ToolRegistry::with_defaults()` starts from a registry containing the `report_progress` tool and nothing else.

A registry plugs directly into the engine-side agent loop as its tool executor, and it validates every call before dispatch. Arguments are checked against the registered JSON Schema; on mismatch the model receives a structured tool error with code `invalid_tool_arguments`, the tool name, and a list of issues, and the tool body is never invoked. A registered schema that is itself invalid is a configuration error raised before any dispatch. Calling a name that is not registered yields the error "Tool not found: <name>".

## The capability contract: ids, references, specs, and `IntoCapability`

### References and the five input forms

A capability, as chapter 01 introduced it, bundles tool definitions with the prompt text that teaches the model when to use them. You attach one to an agent with a single `.capability(...)` call rather than adding tools one at a time. The simplest form is a plain string id.

````rust
let agent = Agent::builder()
    .instructions("Tell the user the time when asked.")
    .model(Model::simulated("It is noon."))
    .capability("current_time")
    .build()?;
````

The string `"current_time"` names the built-in that exposes the `get_current_time` tool; after building, inspecting a session shows that tool among the session's tools. A bare string literal becomes a reference with default configuration. Owned and borrowed `String` values, such as ids loaded from a config file, are accepted the same way.

The explicit form is `CapabilityRef`, which pairs a stable string id with a per-agent JSON configuration object. This is the escape hatch when the id and configuration arrive at run time from a database or a plugin catalog.

````rust
use everruns::CapabilityRef;
use everruns::capability::serde_json::json;

let agent = Agent::builder()
    .instructions("Use the vendor search index.")
    .model(Model::simulated("done"))
    .capability(CapabilityRef::new("current_time"))
    .capability(CapabilityRef::with_config("web_fetch", json!({ "timeout_ms": 30000 })))
    .capability(CapabilityRef::new("vendor.search").config(json!({ "index": "products" })))
    .build()?;
````

`CapabilityRef::new(id)` sets the configuration to an empty JSON object. `CapabilityRef::with_config(id, json)` sets it in one call, and `.config(json)` chains it onto a reference. The `json!` macro comes from `serde_json`; with the default-enabled `capabilities` feature it is reachable as `everruns::capability::serde_json` without a dependency of your own. A reference serializes as `{"ref": "<id>", "config": {...}}`, which is also the shape an agent definition stores for each attached capability.

The dotted id `vendor.search` attaches a capability implemented elsewhere without importing its type. Referencing a third-party or not-yet-implemented id still builds and runs the agent offline: the unknown id is retained as a reference but contributes no prompt text and no tools until a host or plugin provides the implementation.

`.capability(...)` accepts a typed built-in value such as `CompactionConfig::new().budget_percent(0.85)` or `ToolSearch::automatic()`, a code-defined `Definition`, a dynamic `CapabilityRef`, a third-party type implementing `IntoCapability`, or a plain string id. Every form normalizes to a `CapabilitySpec`, which always activates exactly one capability reference and may additionally include a code-defined definition the host must register. A pre-built spec passes through unchanged. `CapabilityRef`, `CapabilitySpec`, and `IntoCapability` are exported from `everruns` with no feature gate.

Several `.capability(...)` calls chain on one builder and compose additively. The typed built-ins `CompactionConfig` and `ToolSearch` are the subject of chapter 07; here they serve only as examples of the typed form.

### Implementing `IntoCapability`

To let your own application type be passed directly to `.capability(...)`, implement `IntoCapability`. The trait is public and non-sealed. A typical implementation builds a `CapabilityRef` with config drawn from the struct's fields and converts it with `.into()`.

````rust
use everruns::{CapabilityRef, CapabilitySpec, IntoCapability};
use everruns::capability::serde_json::json;

struct VendorSearch {
    index: String,
}

impl IntoCapability for VendorSearch {
    fn into_capability(self) -> CapabilitySpec {
        CapabilityRef::new("vendor.search")
            .config(json!({ "index": self.index }))
            .into()
    }
}

let agent = Agent::builder()
    .instructions("Search the index.")
    .model(Model::simulated("done"))
    .capability(VendorSearch { index: "products".into() })
    .build()?;
````

Third-party capability crates may depend on the neutral `everruns-capability` contract crate alone, which pulls in no engine, registry, store, host, async runtime, HTTP, or database dependency, and `everruns` re-exports the identical `CapabilityRef`, `CapabilitySpec`, and `IntoCapability` types, so applications keep depending only on `everruns`. Provider integration packages expose typed values through the same trait and are attached with the ordinary `.capability(...)` call.

### Id grammar and build-time validation

A capability is named by an open string id, not by a variant in a central enum, so new capabilities are added without framework source edits.

Every id is validated against one shared rule set. An id must be non-empty, at most 128 bytes, start with an ASCII letter or `_`, and contain only ASCII letters, digits, `_`, `-`, `.`, or `:`, and it may not start with the reserved `__everruns_` prefix. Valid examples include `noop`, `current_time`, `vendor.search`, `plugin:plg_0193`, and `_private`. The colon is permitted so that references can be namespaced by kind with the open prefixes `mcp:`, `skill:`, `declarative:`, and `plugin:`.

`CapabilityId::parse(id)` returns a validated id or an invalid-id error. Every contract violation is one of four structured `CapabilityError` variants: invalid id, invalid config, invalid definition, or duplicate. Each prints a stable message quoting the id and the reason, for example `invalid capability id "2fast": capability id must start with a letter or underscore` or `duplicate capability id "current_time"`, and each exposes `id()`, `reason()`, and `is_duplicate()`. Hosts map these onto their own error surfaces instead of parsing strings; the Framework maps them onto `BuildError`.

Building the agent validates every capability id and its configuration. Rejected ids include the empty string, `2fast`, `has space`, `vendor/custom`, and `__everruns_private`, each producing `BuildError::InvalidCapability { id, reason }`. Configuration must be a JSON object; a string, null, array, number, or boolean config is rejected with the reason "capability config must be a JSON object". Built-in capabilities run their own config validators and the failing id is reported, so a compaction budget of `2.0` fails under id `compaction`. A function tool named with the reserved prefix is rejected as an invalid capability, because function tools are privately registered as single-tool capabilities.

Each stable id may be activated at most once on an agent. Duplicates are `BuildError::DuplicateCapability { id }`, never last-write-wins overrides. Aliases resolve to the canonical id before the check, so with the `bashkit` feature enabled, `.capability(CapabilityRef::new("virtual_bash")).capability("bashkit_shell")` fails with id `bashkit_shell`. Function tool names share the same namespace, so `.capability("vendor_lookup")` alongside a function tool named `vendor_lookup` collides. A typed built-in and a string reference to the same built-in collide, as in `.capability(ToolSearch::automatic()).capability("auto_tool_search")`. A code implementation cannot shadow a built-in.

Configuration is redacted from debug output: a reference's `Debug` output renders its config as `<redacted>`, and so does the built agent's. It is still not a secret store, since a host may persist or inspect it, so credentials belong in a provider-owned secret mechanism with only a non-secret handle in the config.

## The core `Capability` trait, `CapabilityRegistry`, and capability contributions

The built-in capabilities and any capability a host registers implement the core `Capability` trait. An application writing ordinary tools uses the macro or the `Definition` type from the next section instead. Read this section when you need to understand what a capability can contribute to an agent, or when you are writing the host that registers them.

### Implementing `Capability`

You define a core capability with just an id, a display name, and a description, optionally overriding `tools()` to contribute tools. Every other method has a default.

````rust
struct SampleData;

impl Capability for SampleData {
    fn id(&self) -> &str {
        "sample_data"
    }

    fn name(&self) -> &str {
        "Sample Data"
    }

    fn description(&self) -> &str {
        "Mounts sample records under /samples and reads them on request."
    }

    fn tools(&self) -> Vec<Box<dyn Tool>> {
        vec![Box::new(ScopedLookup)]
    }

    fn system_prompt_addition(&self) -> Option<&str> {
        Some("Consult /samples before answering questions about sample records.")
    }

    fn dependencies(&self) -> Vec<&'static str> {
        vec!["session_file_system"]
    }

    fn mounts(&self) -> Vec<MountPoint> {
        vec![MountPoint::readonly(
            "/samples",
            MountSource::text_file("id,name\n1,Widget\n"),
            self.id(),
        )]
    }

    fn aliases(&self) -> Vec<&'static str> {
        vec!["samples"]
    }
}
````

The sample contributes four things beyond its tool. `system_prompt_addition` adds static text to the agent's system prompt, automatically wrapped in `<capability id="sample_data">` tags; keep it to guidance that the tool names, descriptions, and parameter schemas do not already give. `dependencies()` names `session_file_system`, whose own contributions are pulled in whenever `sample_data` is selected, even if nobody selected them explicitly. `mounts()` places `/samples` into the session filesystem, which is how a capability ships sample data or documentation. `aliases()` lists legacy ids so the capability can be renamed without breaking persisted agent configs. The tool returned from `tools()` needs no attribution of its own, because tools contributed by a capability are automatically attributed to its id and display name in their definitions.

Static text is not the only prompt path. `system_prompt_contribution` generates content at session start, reading AGENTS.md or scanning skills, with access to the session id, locale, session filesystem, model, and session storage; `system_prompt_preview` returns the static text without running that path. Dynamic content is composed after the base prompt so it does not invalidate provider prefix caches.

Per-agent configuration reaches a capability through the `*_with_config` variants of the contribution methods, which receive the JSON config value and let the tools, prompt, hooks, and MCP servers differ by agent. `config_schema` exposes a JSON Schema for those settings and `config_ui_schema` gives UI hints in the react-jsonschema-form `uiSchema` shape. `validate_config` is the server-side check shared by every write path.

Lifecycle and risk are declared rather than contributed. `status()` is `available`, `coming_soon`, `deprecated`, or `retired`, and the runtime honors it: `coming_soon` and `retired` capabilities are silently skipped so an agent that still references one keeps running, while `deprecated` capabilities keep working fully as surfaces warn. Only `retired` is hidden from catalogs, and the legacy spelling `comingsoon` is accepted on input. `risk_level()` is `low` by default, with `medium` and `high` above it; high-risk capabilities require an org admin to assign.

### Registry and dependency resolution

Capability implementations are registered in a `CapabilityRegistry` by value, boxed, or Arc-wrapped, and a fluent builder assembles one in a single expression.

````rust
let registry = CapabilityRegistry::builder()
    .capability(SampleData)
    .build();

let resolved = resolve_dependencies(&["sample_data".to_string()], &registry)?;
assert_eq!(resolved.resolved_ids, vec!["session_file_system", "sample_data"]);
````

`register`, `register_boxed`, and `register_arc` add implementations; re-registering the same canonical id replaces the earlier implementation. `try_register_arc` is the fallible form that rejects duplicate ids and alias collisions at registration time with a structured `CapabilityError`. The registry answers `get` and `has` by id or alias, reports `canonical_id`, and lists its contents.

`resolve_dependencies` orders a selection into topological order with dependencies before dependents and reports which ids were added implicitly. A circular dependency fails with a message naming the chain, and the resolved set is capped at 100 capabilities. Every contribution, from prompt sections to mounts, then appears in selection order with dependencies first, which is why the `session_file_system` tools precede `sample_data` above. When the same capability is provided more than once with config, the last explicit config wins.

The one-call application step is `apply_capabilities`, an async function that takes a base runtime agent (the engine-side agent value), an ordered list of capability ids, the registry, and a system prompt context, and returns the merged agent together with a populated tool registry and the list of applied ids. The core crate exports the registry, the collection and application functions, dependency resolution, and the related types from one place.

### Mounts and declarative capabilities

A mount point is constructed with a path, an access mode, a content source, and the owning capability id. `MountPoint::readonly(path, source, id)` and `MountPoint::readwrite(path, source, id)` are the shortcuts, and `MountPoint::new` takes the access mode explicitly. The default access mode is read-only, meaning agents cannot modify or delete the files.

````rust
let docs = MountDirectoryBuilder::new()
    .file("readme.txt", "Hello")
    .file("config.json", "{}")
    .dir("nested", MountDirectoryBuilder::new().file("inner.txt", "Nested content"))
    .build();

let mount = MountPoint::new("/docs", MountAccess::ReadOnly, docs, "sample_data");
````

`MountSource::text_file(content)` mounts a single inline text file, `MountSource::binary_file(content)` takes base64 content, and `MountSource::directory(entries)` takes a map of filenames to entries; the builder above is the fluent way to compose that map with nested subdirectories. For a read-only tree served from memory across all sessions with no database rows, build a `VirtualFileTree` and insert text files and directories by absolute path, then wrap it with `MountSource::virtual_tree`. Parent directories are created automatically.

A declarative capability is defined from data alone, with no code: a name, display name, description, system prompt text, and a risk level, plus scoped MCP servers, text file mounts, and skill packages. It is referenced by the stable `declarative:<name>` form or by its plain unique name as a shorthand. Here, unknown `declarative:` and `plugin:` ids matter only because they pass through dependency resolution untouched.

## Advanced code-defined capabilities: `Definition` and `Handler`

Most application tools should use `#[everruns::tool]`. The `capability` module is the next step up, for a reusable package that needs several typed tools, capability-level instructions and metadata, execution context, progress events, or call-scoped cancellation. Use the smallest extension contract that fits the behavior you own; both paths are portable and application-authored. The module is behind the `capabilities` feature on `everruns`, which is part of the default feature set, and is also available through the prelude. A plain `cargo add everruns` therefore already enables it; only a manifest with `default-features = false` needs to name it.

````bash
cargo add everruns
````

### Typed handlers

A typed tool is a plain struct implementing the `Handler` trait, with associated `Input`, `Output`, and `Error` types, a name, a description, and an async `execute` method that receives the typed input and a call context. No macro is involved. Input and output structs derive their JSON Schema through derives re-exported from the `capability` module.

````rust
use everruns::{Agent, BuildError, Model, capability};
use everruns::capability::serde_json::json;

#[derive(capability::Deserialize, capability::JsonSchema)]
#[serde(crate = "everruns::capability::serde")]
#[schemars(crate = "everruns::capability::schemars")]
struct LookupInput {
    sku: String,
}

#[derive(capability::Serialize, capability::JsonSchema)]
#[serde(crate = "everruns::capability::serde")]
#[schemars(crate = "everruns::capability::schemars")]
struct Product {
    sku: String,
    name: String,
    price_cents: u64,
    tags: Vec<String>,
}

struct ProductLookup;

#[capability::async_trait]
impl capability::Handler for ProductLookup {
    type Input = LookupInput;
    type Output = Product;
    type Error = capability::Error;

    fn name(&self) -> &str {
        "lookup_product"
    }

    fn description(&self) -> &str {
        "Look up one product by exact SKU."
    }

    fn display_name(&self) -> Option<&str> {
        Some("Look up product")
    }

    fn hints(&self) -> capability::Hints {
        capability::Hints::default().readonly(true).idempotent(true)
    }

    async fn execute(
        &self,
        input: LookupInput,
        context: capability::Context,
    ) -> Result<Product, capability::Error> {
        context.progress("Searching the product catalog").await;
        if input.sku != "SKU-1" {
            return Err(capability::Error::user("sku_not_found", "No product has that SKU")
                .details(json!({ "sku": input.sku })));
        }
        Ok(Product {
            sku: input.sku,
            name: "Widget".into(),
            price_cents: 1999,
            tags: vec!["sample".into()],
        })
    }
}
````

The two crate-path attributes on each struct point `serde` and `schemars` at the framework's re-exports, so your manifest still lists neither. When you depend on the contract crate directly instead, the paths are `everruns_capability::serde` and `everruns_capability::schemars`.

`Output` here is a structured type, and the model receives typed JSON rather than flattened text. `display_name` is optional and gives clients a readable label for `lookup_product`. `hints` is optional too; the `Hints` builder offers the four MCP hints and `long_running` from the wire types section, plus a `concurrency_class` whose non-empty key makes calls sharing it execute sequentially, and a `metadata` blob. The error type may be your own enum as long as it converts into `capability::Error`.

### Assembling a `Definition`

`Definition::new(id, name, description)` packages typed tools into one reusable capability. The id is the stable persisted identifier; the name and description are human-facing catalog text.

````rust
let catalog = capability::Definition::new(
    "product_catalog",
    "Product catalog",
    "Application-owned product data.",
)
.instructions("Use exact SKUs. Never invent catalog entries.")
.metadata(json!({ "owner": "commerce" }))
.tool(ProductLookup);

let agent = Agent::builder()
    .instructions("Answer product questions from the catalog.")
    .model(Model::simulated("Widget costs $19.99."))
    .capability(catalog.clone())
    .build()?;
````

`.instructions(...)` adds behavioral guidance to the agent's system prompt whenever the capability is installed. Use it for cross-tool ordering and constraints rather than repeating tool names and schemas; it is separate from the agent's global instructions. `.metadata(json)` attaches host-owned JSON the engine does not interpret, and because it may be persisted or shown to clients the no-credentials rule from the wire types applies to it as well. `.tool(handler)` adds a handler, and tools are kept in registration order.

A definition is an immutable, cloneable value built independently of any agent, which is why `catalog.clone()` above installs the same capability on this agent and leaves `catalog` free for another. Authors never touch an engine registry; the host registers the definition privately at session start. A definition passes directly to `.capability(...)` or converts with `.into()`. Its `id()`, `name()`, `description()`, `instructions_text()`, `metadata_value()`, and `tools()` can all be read back, and every tool exposes a stable protocol descriptor through `spec()` with its name, display name, description, generated input and output schemas, and hints. Those schemas are available before the capability is installed, for tests or for a host catalog.

````rust
let spec = catalog.tools()[0].spec();
assert_eq!(spec.name(), "lookup_product");
assert_eq!(spec.input_schema()["required"], json!(["sku"]));
assert_eq!(spec.hints().readonly, Some(true));
````

The advanced surface exports no stores, registries, provider credentials, or filesystem backends of its own, so inject application-owned clients or state by storing them in the handler struct when you construct the definition.

### Compile-time and build-time checks

A handler's output type must implement `Serialize` and `JsonSchema`. A type that lacks either fails to compile with a trait-bound error pointing at the `Output` associated type.

````rust
struct NotSerializableOrSchema;

struct Broken;

#[capability::async_trait]
impl capability::Handler for Broken {
    type Input = LookupInput;
    type Output = NotSerializableOrSchema;
    type Error = capability::Error;
    // name, description, execute ...
}
````

````text
error[E0277]: the trait bound `NotSerializableOrSchema: JsonSchema` is not satisfied
error[E0277]: the trait bound `NotSerializableOrSchema: serde::Serialize` is not satisfied
````

Both errors point at the `type Output` line, and the notes name the bound in `everruns::capability::Handler::Output`. For the `Serialize` half the compiler suggests adding `#[derive(serde::Serialize)]` for a local type or checking for a `serde` feature flag on a foreign crate.

`Definition::validate()` checks a definition before install. It rejects an invalid id, a blank name, a blank description, blank instructions, zero tools with the reason "capability must define at least one tool", and any tool with a blank description, returning an invalid-definition error with the id and reason. At build time the same validation runs and the definition's tool names and schemas are checked; a definition with no tools fails the build as `BuildError::InvalidCapability`, and the once-per-agent id rule from the capability contract applies to a definition's id and tool names alike.

````rust
let err = Agent::builder()
    .instructions("Use the tools.")
    .model(Model::simulated("ok"))
    .capability(capability::Definition::new("empty", "Empty", "No tools."))
    .build()
    .expect_err("a definition needs a tool");
assert!(matches!(err, BuildError::InvalidCapability { .. }));
````

## Handler `Context`, progress, cancellation, structured errors, and the examples

### What the context tells a handler

Inside `execute`, the call context exposes `tool_name()`, an opaque `session_id()` for correlation and application-side scoping, `workspace_id()`, and `locale()`, the resolved BCP 47 locale when the host supplied one. `context.progress("...").await` emits a best-effort progress message that surfaces as a correlated `tool.progress` session event so the host can show status while the tool works; delivery failure never fails the tool.

`context.cancellation()` exposes call-scoped cancellation with an `is_cancelled()` check and an awaitable `cancelled()` future. Ordinary awaited work needs no cancellation branch, because cancelling a turn drops the handler future. Clone the cancellation only into child tasks or watchers that might survive after `execute` is dropped. The signal fires on cancellation and on every other completion path, including normal success, so the child work always ends with the call.

````rust
async fn execute(
    &self,
    input: LookupInput,
    context: capability::Context,
) -> Result<Product, capability::Error> {
    let cancellation = context.cancellation().clone();
    tokio::spawn(async move {
        cancellation.cancelled().await;
        // stop the watcher this handler started
    });
    context.progress("Looking up the forecast").await;
    // ... awaited work needs no cancellation check
    Ok(product_for(&input.sku))
}
````

### Structured errors

`Error::user(code, message)` returns a structured, user-visible error, and `.details(json)` attaches structured data. The model receives a JSON payload with `code`, `message`, and `details`, so it can act on a stable code such as `sku_not_found` rather than parsing prose. `Error::internal(code, message)` is the handler form of the redacted internal channel; its code, message, and details are logged rather than shown. An error exposes `code()`, `message()`, `details_value()`, and `visibility()`, and prints as `[code] message`.

Two codes are produced automatically. Arguments that do not match the declared input schema are rejected as a user-visible error with code `invalid_arguments` and a message that the tool arguments did not match the declared schema. A result that fails to serialize becomes an internal error with code `result_serialization`, hidden from the model.

### Testing without an engine

A context can be built with no runtime at all from a tool name, a session id, and a workspace id. Its defaults are inert: progress is a no-op and cancellation never fires. A tool in a definition can then be executed directly from raw JSON arguments.

````rust
let tool = &catalog.tools()[0];

let context = capability::Context::new("lookup_product", "session-1", "workspace-1");
let product = tool.invoke(json!({ "sku": "SKU-1" }), context).await?;
assert_eq!(product["name"], "Widget");

let context = capability::Context::new("lookup_product", "session-1", "workspace-1");
let error = tool.invoke(json!({ "sku": 7 }), context).await.unwrap_err();
assert_eq!(error.code(), "invalid_arguments");
````

`with_locale`, `with_progress_sink`, and `with_cancellation_signal` chain onto a test context to attach a locale, a recording progress sink, or a controllable cancellation signal. The `ProgressSink` trait routes progress messages into your own event stream, and the matching `CancellationSignal` trait exists for alternate runtimes; the Framework implements both against its session event stream and turn cancellation.

### The examples

Two shipped examples exercise this surface. `advanced_capability` builds a custom capability against a real provider and demonstrates the typed protocol, metadata, progress, and structured errors. It requires the `capabilities` and `openai` features, reads the provider key through `OpenAI::from_env()`, and therefore needs `OPENAI_API_KEY` in the environment.

````bash
cargo run -p everruns --features capabilities,openai --example advanced_capability
````

`capability_configuration` runs fully offline with only the `capabilities` feature. It shows one `.capability(...)` entrypoint accepting typed compaction and tool-search values, a code-defined definition, and a dynamic vendor reference on the same builder.

````bash
cargo run -p everruns --features capabilities --example capability_configuration
````

*2026-09-17 03:24 - claude-fable-5.1*
