<!-- source: everruns/everruns @ 6cf2c15e5 (crate everruns 0.20.0). The working tree fast-forwarded to 7d9e07a0b (0.21.1) during extraction; identifiers and examples were audited against 6cf2c15e5. -->
# Testing, Plugins, and Extension

Every chapter before this one ran its examples against `Model::simulated`, and none of them explained why that was safe to rely on. The answer is that the simulator is not a mock: it travels the normal provider resolution and execution path and only the reply is produced locally. This chapter works outward from that first offline test to the published contracts that let code outside the `everruns` repository, from a plugin directory to a hand-assembled host, become part of an agent.

## Offline testing with the simulated model and test fixtures

### A complete offline test

````rust
use everruns::prelude::*;

#[tokio::test]
async fn agent_follows_the_application_flow() -> Result<(), Box<dyn std::error::Error>> {
    let agent = Agent::builder()
        .instructions("Return only the answer.")
        .model(Model::simulated("4"))
        .capability("session_file_system")
        .build()?;

    let engine = Engine::new();
    let session = engine.create(agent);

    let context = session.inspect().await?;
    assert!(context.tools.iter().any(|tool| tool.name == "read_file"));

    let turn = session.send_and_wait("What is 2 + 2?").await?;
    assert!(turn.success);
    assert_eq!(turn.response, "4");
    Ok(())
}
````

`Model::simulated("4")` is the default testing tool, and a passing offline test exercises the same code as a live run without opening a network connection or needing a key. `build()` returns a `Result`, which is why a misconfigured agent surfaces as an error the test can report instead of a panic. `session.inspect()` shows the resolved tool list before any turn runs, and enabling `session_file_system` makes `read_file` appear in it. `session.send_and_wait` sends the message, blocks until the turn completes, and returns a `Turn` whose `success` and `response` fields you assert on.

The function is an ordinary `#[tokio::test]` returning `Result<(), Box<dyn std::error::Error>>`, which lets `?` work on every framework error, and the `tokio` dev-dependency needs only its `macros` and `rt-multi-thread` features. The simulator is safe to ship in production binaries because it comes from the publishable `everruns-llmsim` crate.

### Scripting tool calls and draining events

````rust
use everruns::{Agent, FunctionTool, InMemoryEngine, LlmSimConfig, Model, ToolCall};
use serde_json::json;

let lookup = FunctionTool::new(
    "lookup",
    "Look up a value.",
    json!({ "type": "object", "properties": { "key": { "type": "string" } } }),
    |arguments: serde_json::Value| async move { Ok::<_, String>(json!({ "value": arguments["key"] })) },
);

let model = Model::simulated_with_config(LlmSimConfig::fixed("done").with_tool_call_sequence(vec![
    vec![ToolCall { id: "call_lookup_1".into(), name: "lookup".into(), arguments: json!({ "key": "answer" }) }],
    vec![],
]));

let agent = Agent::builder()
    .instructions("Use lookup, then report the result.")
    .model(model)
    .tool(lookup)
    .build()?;

let session = InMemoryEngine::new().create(agent.clone());
let mut stream = session.events();
let result = session.run("hello").await;
drop(session);

let mut observed = Vec::new();
while let Some(event) = stream.recv().await? {
    observed.push(event);
}
let turn = result?;
assert_eq!(turn.tool_calls, 1);
assert_eq!(turn.response, "done");
````

`LlmSimConfig::fixed(final_text).with_tool_call_sequence(tool_call_rounds)` scripts the tool calls and the final answer in one configuration, and `Model::simulated_with_config(sim)` turns it into a model. Each inner list is one model response. A list with one `ToolCall` makes the model call that tool, and the empty list ends the loop with the fixed final text, so the test can assert exact iteration and tool call counts. The facade's own tests reach the same shape through a `Model::simulated_scripted(final_text, tool_call_rounds)` shorthand, but that helper is `pub(crate)` and is not available to application code. `FunctionTool::new(name, description, json_schema, handler)` defines the tool inline, which keeps an event-flow test free of a separate tool module. The event half runs on `InMemoryEngine`, a source-compatible alias for `everruns::Engine` rather than a second implementation, and follows a fixed order: subscribe with `session.events()` before running, run the turn, drop the session to close the stream, then drain the receiver.

`LlmSimConfig` is re-exported from the facade and included in its prelude, and the same `Model::simulated_with_config(sim)` entry point accepts every other simulator control covered in the next section: multiple assistant turns, injected provider errors, delays, and request capture.

An application test suite has a recommended shape, and every part of it except the last runs offline:

- Build-time validation tests for agent, tool, model, MCP, and compaction configuration that never run a session. The bashkit-repo-agent example's `builds_without_contacting_the_provider` asserts that `build(..)` with `OpenAI::new("test-key")` is `Ok`.
- Offline session tests with `Model::simulated`.
- Context assertions through `Session::inspect`.
- Event and cancellation tests through the public session API; live observation and cancellation never require ownership of the host event bus or task runner.
- A small opt-in live-provider suite for protocol integration.

Keep normal tests credential-free and deterministic, and never make a live model's wording or tool choice a unit-test oracle. Run workspace and local-state tests in temporary directories so they never touch developer data. The facade's own acceptance tests use `trybuild` for compile-fail assertions on the tool macro and `tempfile` for temporary workspaces and plugin fixtures. They also pull in `everruns-test-support` with its `fixtures` feature. Use `everruns-test-support` only for testing and demo helpers such as its in-memory agentic loop, writable fixtures, test doubles, and fake capabilities; its simulator re-exports exist only as a 0.18 migration bridge for 0.17 import paths, and new code imports the simulator from the facade or from `everruns-llmsim`. Advanced hosts test deterministically by pairing the simulator's `host` feature with those fixtures, with mocked HTTP available through `wiremock`.

## Scripting the llmsim simulator with replies, tool calls, delays, and failures

### Response modes and scripted turns

````rust
use everruns::{Agent, LlmSimConfig, Model};
use everruns_llmsim::{SimToolCall, SimTurn};

let simulation = LlmSimConfig::scripted(vec![
    SimTurn::ToolCalls(vec![SimToolCall {
        name: "lookup".into(),
        arguments: serde_json::json!({"id": 7}),
        id: Some("call_lookup".into()),
    }]),
    SimTurn::Assistant("approved".into()),
]);

let agent = Agent::builder()
    .instructions("Use lookup, then report the result.")
    .model(Model::simulated_with_config(simulation))
    .build()?;
````

`LlmSimConfig` comes from the facade; `SimTurn` and `SimToolCall` come from `everruns_llmsim` directly. `LlmSimConfig::scripted(turns)` replays the list in order, one entry per model call. A `SimToolCall` that omits `id` receives a deterministic generated id of the form `call_llmsim_{turn}_{call}`; the first call of the first turn is `call_llmsim_0_0`. A script with zero turns fails with the configuration error "llmsim scripted config must contain at least one turn" rather than panicking.

A `SimTurn` is one of `Assistant(String)`, `ToolCalls(Vec<SimToolCall>)`, `Mixed { text, tool_calls }`, `Error(SimError)`, or `StreamStall`. The error variant injects a typed provider failure on that turn. `SimError::RateLimit` maps to `LlmErrorKind::RateLimited`; `Timeout`, `Transport`, and `Overloaded` map to `Unavailable`; `Authentication` and `QuotaExhausted` map to their namesakes; `UnsupportedModel(String)` becomes a model-not-available error; `InvalidResponse(String)` maps to `InvalidRequest`; `Other(String)` maps to `Other`, and a test asserts on the mapped kind with `err.is_rate_limited()`. When a script runs out, `.with_on_exhausted(mode)` chooses between `OnExhausted::RepeatLast` (the default), `OnExhausted::Loop`, and `OnExhausted::Error`.

Simpler response modes:

- `LlmSimConfig::fixed(text)` returns one canned reply on every call. With no configuration at all the reply is "Hello! I'm a simulated LLM response."
- `LlmSimConfig::echo()` replies with "Echo: " followed by the newest user message.
- `LlmSimConfig::sequence(vec)` advances one entry per call and cycles when exhausted; the fourth call of a three-entry list returns the first entry again.
- `LlmSimConfig::lorem(target_tokens)` generates filler text of roughly the requested token count.
- `LlmSimConfig::error(message)` makes every call fail with "LLM error: {message}" before any generation.
- `LlmSimConfig::model_not_available()` makes every call fail with a model-not-available error for the requested model.

### Tool calls, latency, capture, and registration

- `.with_tool_calls(vec)` attaches the same set of tool calls to every response; an empty list means none.
- `.with_latency()` simulates time to first token and inter-token delays; it is off by default so tests stay fast. A model name containing `-latency` switches it on per request, and a model name of the form `llmsim-ttft-2000` requests a two-second first-token delay unless a configured `response_delay` takes precedence.
- `.with_response_delay(Duration)` delays every reply by a fixed duration.
- `.with_model(name)` sets the name the simulator uses internally and in Debug output (default `llmsim-model`); completion metadata still reports the model from the request's call configuration.
- `.with_response_id(id)` includes a response id in completion metadata for tests that check `previous_response_id` chaining.
- `.with_effort_capture(sink)` records the per-call reasoning effort the driver observed, in call order, even on error turns.
- `.with_message_capture(sink)` records the exact provider-visible message list of every call; assert on it to see what survived context assembly and trimming.

Every knob is also a public field, so a struct literal can build the configuration. Two settings are reachable only that way: an empty text response for tool-only turns, and conditional tool calls built from `ToolCallPattern::new(contains, tool_calls)`, which fire only when a user message contains the substring. Matching scans user messages newest first so injected notifications do not mask the trigger.

The reply arrives as a token-by-token stream in provider order: reasoning, text deltas, tool calls, then a done event whose metadata carries token counts, the model, and finish reason "stop". When a call requests reasoning, the simulator emits the readable delta `LLMSIM_REASONING_TEXT` ("llmsim deliberating about the request") followed by an opaque reasoning item with a fixed signature and encrypted payload; the split lets a test verify that reasoning text reaches the API while replay state does not.

Outside the facade, `LlmSimDriver::new(config)` constructs the driver directly and `create_chat_driver(config)` returns a boxed chat driver for tests. `register_driver(&mut registry)` registers the simulator as the `LlmSim` provider in a `DriverRegistry`; any API key is accepted and the display name is "LLM Simulator". `register_driver_with_config(&mut registry, config)` registers a scripted scenario that servers and workers replay without a key, with counters shared across every driver constructed from it. `llm_sim_provider(config)` builds a standalone `llmsim` provider value for any builder that exposes a raw provider seam, such as `LocalRuntimeBuilder::provider_with_default_model`. Select the model through the constants `LLMSIM_PROVIDER` ("llmsim") and `LLMSIM_MODEL_ID` ("llmsim-model") rather than hard-coded strings. Four prebaked scripts ship with the crate: `auditor_demo_script()`, `guarded_bash_demo_script()`, `session_tasks_demo_script()`, and `monitor_demo_script()`. The hook-bundles example selects one through the `LLMSIM_DEMO` environment variable; without it the simulator returns its default canned response and never calls a tool.

## Plugin manifests, the plugin compiler, and declarative contributions

### Loading a plugin directory

````rust
use everruns::prelude::*;

let agent = Agent::builder()
    .instructions("Follow the plugin's guidance for release work.")
    .model(Model::simulated("ready"))
    .plugin("./plugins/acme-tools")?
    .build()?;
````

`Agent::builder().plugin(path)` reads and compiles the directory immediately and returns `Result<Self, PluginError>`, hence the `?` inside the chain. Invalid or unsafe plugin input fails before the agent is built. `PluginError` is typed and comparable, exposes the failing directory through `path()`, displays as "plugin {path}: {message}", and implements the standard error trait. Non-fatal warnings are kept and returned later through `Session::inspect`. The loaded plugin contributes instructions and capabilities under the id `plugin:{name}`, using the manifest name for a standalone plugin and the installation public id for a server-managed one. Directories are size-limited by the public constants `MAX_PLUGIN_FILES`, `MAX_PLUGIN_FILE_BYTES`, and `MAX_PLUGIN_TOTAL_BYTES`, and the loader rejects path traversal and symlinks.

````text
acme-tools/
  plugin.json
  agents/
    reviewer.md
  skills/
    release-notes/
      SKILL.md
      nested/readme.txt
  commands/
    ms-docs.md
  mcp.json
  assets/
    icon.svg
````

````json
{
  "$schema": "https://agent-plugins.org/schemas/1.0.0/plugin.schema.json",
  "name": "acme-tools",
  "description": "Guidance and tools for Acme release work.",
  "icon": "./assets/icon.svg"
}
````

A manifest at `plugin.json` with a `$schema` key marks the portable Agent Plugins v1 dialect; a manifest at `.claude-plugin/plugin.json` marks the legacy host dialect, and the compiler applies the matching rules. A v1 name is 1 to 64 characters of lowercase letters, digits, `-`, and `.`, starting and ending alphanumeric, with no `--` or `..`; a `$schema` other than the 1.0.0 plugin schema is rejected. A legacy name must fit in 43 bytes, must start with a lowercase letter, may contain only lowercase letters, digits, `-`, and `_`, and cannot end with `-` or `_`. A legacy manifest must have a non-empty `description`; a v1 manifest may omit it and receives "Agent plugin {name}". The `icon` key points at a relative `.svg` inside the plugin and is embedded as a base64 data URI; a missing, absolute, external, or unsafe SVG falls back to the `puzzle` icon with a warning.

Each Markdown file in `agents/` becomes one `<agent name="..." description="...">` section of a combined system prompt, in filename order, with `name:` and `description:` read from YAML frontmatter between `---` markers. Each `skills/<name>/SKILL.md` becomes a capability skill; text reference files beside it ship with the skill, and binary files are skipped with a warning. Each Markdown file in `commands/` compiles to a user-invocable skill. The manifest keys `agents`, `skills`, and `commands` override those locations as a single path or a list of paths. A v1 plugin declares remote MCP servers in a root `mcp.json` with `$schema` plus an `mcpServers` object using `streamable-http` transport, and declares OAuth through the manifest key `extensions.com.everruns.mcpServers.<name>.auth` set to `oauth`. A legacy plugin uses a root `.mcp.json` or the manifest key `mcpServers`, pointing at one file, a list of files, or an inline map where each server takes a `url`, string `headers`, and an `auth` of `oauth` or `none`. Non-loopback remote URLs must use HTTPS. Stdio transport is skipped, and `api_key` auth and `oauth_provider_id` are dropped, each with a warning, because a plugin package cannot carry key material or spawn processes. Unsupported manifest fields such as `hooks` and `lspServers` produce "is not supported in v1 and will be ignored" warnings.

### The plugin compiler and declarative definitions

````rust
use everruns_core::plugins::{PluginFileSet, compile_plugin};

let files = PluginFileSet::from_dir(std::path::Path::new("./plugins/acme-tools"))?;
let compiled = compile_plugin(&files)?;
for warning in &compiled.warnings {
    eprintln!("{warning}");
}
let definition = compiled.definition;
````

`compile_plugin` is the function behind `.plugin(path)`. It returns a `CompiledPlugin` carrying the parsed `manifest` and the compiled `definition`, plus a `warnings` list. `PluginFileSet::from_dir` loads a directory and `PluginFileSet::from_map(name, files)` builds the same unit from an in-memory file map.

````json
{
  "name": "research_pack",
  "description": "Primary-source research guidance.",
  "system_prompt": "Read sources before citing them.",
  "dependencies": ["session_file_system"],
  "files": [{ "path": "/policy/sources.md", "content": "Cite primary sources only." }]
}
````

The compiled definition is a `DeclarativeCapabilityDefinition`, and you can write one directly without a plugin. Only `name` and `description` are required; `status` defaults to available, `risk_level` to low, `icon` to `puzzle`, and `category` to "Declarative". Names are at most 38 bytes with the same lowercase-letter grammar as legacy plugins, descriptions are capped at 512 bytes, the `system_prompt` at 64 KiB, and the `display_name` at 80 bytes, all measured in UTF-8 bytes. A definition may attach up to 16 `mcp_servers` and 16 `skills`. It may mount up to 32 `files` of 64 KiB each; files use absolute mount paths, default to read-only, and cannot sit under `/.agents/skills`, which is reserved for skills. `dependencies` may name built-in or plugin capabilities such as `session_file_system` or `plugin:tools`, never other declarative capabilities. `validate_declarative_capability_definition` returns a specific human-readable error for the first violated rule, and the plugin compiler runs every compiled definition through it. A definition is referenced as `declarative:{name}` and a compiled plugin as `plugin:{name}`; either reference carries the full definition inside the per-agent configuration, so no registry entry is required and hydration discards unknown or overriding keys.

## Capability, provider, and builtin packs

### A capability pack on the neutral contract

````rust
use everruns_capability::definition as capability;
use everruns_capability::definition::{Context, Definition, Error, Handler, Hints};
use everruns_capability::{CapabilityRef, CapabilitySpec, IntoCapability};

#[derive(capability::Deserialize, capability::JsonSchema)]
#[serde(crate = "everruns_capability::serde")]
#[schemars(crate = "everruns_capability::schemars")]
pub struct AddInput { pub a: i64, pub b: i64 }

#[derive(capability::Serialize, capability::JsonSchema)]
#[serde(crate = "everruns_capability::serde")]
#[schemars(crate = "everruns_capability::schemars")]
pub struct AddOutput { pub sum: i64 }

pub struct Add;

#[capability::async_trait]
impl Handler for Add {
    type Input = AddInput;
    type Output = AddOutput;
    type Error = Error;

    fn name(&self) -> &str { "external_add" }
    fn description(&self) -> &str { "Add two integers using the external capability pack." }
    fn hints(&self) -> Hints { Hints::default().readonly(true).idempotent(true) }

    async fn execute(&self, input: Self::Input, context: Context) -> Result<Self::Output, Error> {
        context.progress("adding").await;
        Ok(AddOutput { sum: input.a + input.b })
    }
}

pub fn math_pack() -> Definition {
    Definition::new("external_math", "External Math", "Typed math tools from an out-of-workspace capability crate.")
        .instructions("Use external_add for exact integer sums.")
        .tool(Add)
}

pub struct VendorSearch { pub index: String }

impl IntoCapability for VendorSearch {
    fn into_capability(self) -> CapabilitySpec {
        CapabilityRef::new("vendor.search")
            .config(capability::serde_json::json!({ "index": self.index }))
            .into()
    }
}
````

`everruns-capability` is the pack's only dependency. It imports nothing from the facade, from `everruns-core`, from `everruns-host`, or from Tokio, and an application consumes it with ordinary imports and no central enum edit anywhere. The input and output structs derive their JSON schemas through the contract crate's re-exports, which spares the pack a direct serde or schemars dependency. A `Handler` supplies its own stable model-facing `name` and `description` plus optional `hints`. Its async `execute(input, context)` can report progress and returns a typed output the framework serializes. `Definition::new(id, display_name, description)` bundles the tools with shared model instructions; adding `.capability(math_pack())` to an agent makes `external_add` appear in the session tool list. `VendorSearch` shows the other shape, a plain struct whose fields become consumer-supplied configuration. `CapabilityRef::new("vendor.search").config(json)` references a dynamic capability by string id plus a JSON config object. Only the id grammar and the JSON object boundary are validated centrally; the referenced implementation owns the inner schema. The persisted shape is `{"ref": "vendor.search", "config": {"index": "prod"}}`, config payloads never appear in Debug output, and application clients keep secrets outside that serialized configuration altogether, with Brave Search as the reference integration.

A definition is testable before it ships. `pack.validate()` checks it programmatically, and `pack.tools()[0].spec()` exposes each tool's `name()`, `input_schema()`, `output_schema()`, and `hints()` without running an agent.

### A builtin pack

````rust
use everruns_builtins::register_portable_capabilities;
use everruns_core::CapabilityRegistry;

let mut registry = CapabilityRegistry::new();
register_portable_capabilities(&mut registry)
    .expect("a fresh registry cannot collide with the portable catalog");
assert!(registry.has("compaction"));
assert!(registry.has("auto_tool_search"));
assert!(registry.has("tool_call_repair"));
assert!(!registry.has("web_fetch"));
````

`register_portable_capabilities(&mut registry)` installs the framework's portable built-in policy bundle into a registry you own, importing only the builtins crate and the core registry type. It returns a `Result` that fails on an id collision with an already-registered capability. The portable catalog leaves out host-specific capabilities such as `web_fetch` and `platform` by design. `CapabilityRegistry::new()` and `DriverRegistry::new()` start empty, and registration is additive. Integration crates can also register capabilities at link time by submitting an `IntegrationPlugin` (with an `experimental_only` flag and an optional `feature_flag` resolved fail-closed from a `FEATURE_<NAME>` environment variable), which a host pulls in under its own policy with `register_inventory_plugins`.

### A provider pack

````rust
use async_trait::async_trait;
use everruns_provider::driver_registry::{
    ChatDriver, DriverRegistry, LlmCallConfig, LlmCompletionMetadata, LlmMessage, LlmResponseStream, LlmStreamEvent,
};
use everruns_provider::runtime_provider::{Provider, ProviderEndpoint};

pub const ACME_DRIVER_ID: &str = "acme_llm";

pub struct AcmeChatDriver;

#[async_trait]
impl ChatDriver for AcmeChatDriver {
    async fn chat_completion_stream(
        &self,
        _endpoint: &ProviderEndpoint,
        messages: Vec<LlmMessage>,
        _config: &LlmCallConfig,
    ) -> everruns_provider::Result<LlmResponseStream> {
        let last = messages.last().map(|message| message.content.to_text()).unwrap_or_default();
        let events = vec![
            Ok(LlmStreamEvent::TextDelta(format!("acme:{last}"))),
            Ok(LlmStreamEvent::Done(Box::new(LlmCompletionMetadata::default()))),
        ];
        Ok(Box::pin(futures::stream::iter(events)))
    }
}

pub fn register_driver(registry: &mut DriverRegistry) {
    registry.register_external(ACME_DRIVER_ID, |config| {
        Provider::new(config.provider.clone(), AcmeChatDriver)
            .base_url(config.base_url.as_deref().unwrap_or("https://acme.test"))
            .into_boxed_driver()
    });
}
````

`chat_completion_stream` is the entire `ChatDriver` surface you implement. It receives the endpoint and the call configuration along with the message history, reads incoming text through the message content's text accessor, and streams back text deltas followed by a `Done` event carrying completion metadata over any `futures` stream. The crate depends on the provider SPI only, plus the `async-trait` crate for the attribute. `register_external` puts the driver in a `DriverRegistry` under a string wire id through the same descriptor machinery the official provider crates use, with no vendor branching inside the SPI. Inside the closure, `Provider::new(config.provider, driver)` wraps the driver and `.base_url(..)` applies a fallback when the config has none. `.into_boxed_driver()` then produces the boxed value the registry stores.

An externally registered driver is addressed as `DriverId::external(ACME_DRIVER_ID)` and checked with `registry.has_driver(&id)`. A bare `ProviderConfig::new(id)` with no API key instantiates it through `registry.create_chat_driver(..)`, so credential-free providers need no dummy secrets. The non-streaming `driver.chat_completion(&ProviderEndpoint::default(), vec![LlmMessage::text(LlmMessageRole::User, "ping")], &call_config).await` convenience works too, where `call_config` is an `LlmCallConfig` whose `model` is `"acme-1"`, and returns `response.text == "acme:ping"`, because the SPI aggregates the deltas.

## Event log SPI and workspace provider seam

### Implementing a custom event log

````rust
use async_trait::async_trait;
use everruns_core::events::{Event, EventRequest};
use everruns_host::{EventCursor, EventDurability, EventLog, EventLogError, EventPage, EventReadRequest, EventReader};
use everruns_provider::typed_id::EventId;

#[async_trait]
impl EventReader for ExternalEventLog {
    async fn read_page(&self, request: EventReadRequest) -> Result<EventPage, EventLogError> {
        let session_id = request.session_id();
        let current_high = self.high_watermark(session_id);
        let (after, snapshot) = match request.cursor() {
            None => (0, current_high),
            Some(cursor) => {
                if cursor.session_id() != session_id {
                    return Err(EventLogError::CrossSessionCursor { detail: "cursor belongs to another session".into() });
                }
                match cursor.snapshot_high_watermark() {
                    Some(snapshot) if snapshot > current_high => {
                        return Err(EventLogError::ExpiredCursor { detail: "cursor snapshot is not available".into() });
                    }
                    Some(snapshot) => (cursor.after_sequence(), snapshot),
                    None => (cursor.after_sequence(), current_high),
                }
            }
        };
        let limit = request.limit().get();
        let mut events = self.events_in(session_id, after, snapshot, limit + 1);
        let has_more = events.len() > limit;
        events.truncate(limit);
        let last = events.last().and_then(|event| event.sequence).unwrap_or(after);
        let next_cursor = has_more
            .then(|| EventCursor::continuation(session_id, last, snapshot))
            .transpose()?;
        EventPage::new(events, next_cursor, snapshot)
    }
}

#[async_trait]
impl EventLog for ExternalEventLog {
    async fn append(&self, request: EventRequest) -> Result<Event, EventLogError> {
        if request.is_ephemeral() {
            return Err(EventLogError::InvalidAppend { detail: "ephemeral events are sink-only".into() });
        }
        let sequence = self.next_sequence(request.session_id);
        let event = request.into_event(EventId::new(), sequence);
        self.persist(&event)?;
        Ok(event)
    }

    fn durability(&self) -> EventDurability { EventDurability::CrashDurable }
}
````

Conversation persistence is the one backend with a single write path, and this SPI replaces it. `EventReader` has the single method `read_page`; `EventLog` builds on it and adds `append` and `durability`. Everything the implementation touches is a published public path, which is what lets the log live in your own crate. `high_watermark`, `events_in`, `next_sequence`, and `persist` are your storage; the rest is the contract. Supply the finished log to composition with `HostBackends::with_event_log`.

The log owns event identity. On append it assigns `EventId::new()` and the next per-session sequence, then finalizes with `request.into_event(id, sequence)`. Sequences start at 1 and are unique and strictly increasing per session, but they need not be contiguous: a hidden physical record may consume a sequence and never appear in replay, and readers must accept the gap while preserving order. Ephemeral events are sink-only and must be rejected at append time with `InvalidAppend`. `durability()` returns `EventDurability::CrashDurable` for real storage or `EventDurability::Volatile` for an in-memory log. The log is append-only, with no truncate, rewind, or mutation contract, and history remains a read-only projection.

Reading follows a small contract. Honor the caller's limit through `request.limit().get()` and fetch limit plus one rows to detect more; return a continuation cursor only when more remain. `EventPage::new(events, next_cursor, snapshot)` and `EventCursor::continuation(session, after, snapshot)` validate the shared invariants and return Results. A request with no cursor is an initial read that captures the current high watermark. A cursor with a snapshot is a continuation pinned to that snapshot, so later appends stay invisible and paging neither skips nor duplicates. A cursor without a snapshot, built by `EventCursor::after(session, sequence)`, is a poll that captures a fresh snapshot. The typed errors replace panics: `CrossSessionCursor` for another session's cursor, `ExpiredCursor` when its snapshot is not available, `IncompatibleCursor` when its position exceeds its snapshot, and `InvalidRead` for negative positions or pages containing events beyond the snapshot.

````rust
let first = log.read_page(EventReadRequest::new(session, limit(2))).await?;
let continuation = first.next_cursor.clone().expect("continuation");
let second = log.read_page(EventReadRequest::from_cursor(continuation, limit(2))).await?;
assert!(second.next_cursor.is_none());

let history = EventHistory::new(log.clone())
    .read_page(EventHistoryReadRequest::new(session_id, EventHistoryReadLimit::new(16)?))
    .await?;
````

A user input event request is `EventRequest::new(session_id, EventContext::empty(), InputMessageData::new(Message::user(text)))`, and it becomes an `input.message` event on append. The host's `EventHistory` projection rebuilds message history from any custom log, reading text messages through an optional text accessor so non-text messages are skipped. Narrower seams sit beside the log. `EventSink` forwards emitted events to your own destination through one `try_send` method, and `NoopEventSink` discards them. The host emits through `HostEventEmitter` and reports `EventDeliveryStats`. `EventEmitter` is a single async `emit` method that routes events from the agent loop through your observer. `SessionFileSystemFactory` plugs in a session filesystem backend with a stable `name` and an async `create_session_file_system` that returns a filesystem trait object.

### Implementing a workspace provider

````rust
use std::sync::Arc;
use async_trait::async_trait;
use everruns_host::{
    InMemorySessionFileStore, WorkspaceBinding, WorkspaceDescriptor, WorkspaceError, WorkspaceHeadDescriptor,
    WorkspaceHeadId, WorkspaceHeadRequest, WorkspaceHeadResource, WorkspaceId, WorkspaceProvider, WorkspaceProviderId,
};

#[async_trait]
impl WorkspaceProvider for ExternalWorkspaceProvider {
    fn id(&self) -> WorkspaceProviderId {
        WorkspaceProviderId::new("example.external-workspace").unwrap()
    }

    async fn open_workspace(&self, locator: &str) -> Result<WorkspaceDescriptor, WorkspaceError> {
        Ok(WorkspaceDescriptor { id: WorkspaceId::from_seed(7), name: locator.to_owned(), metadata: Default::default() })
    }

    async fn open_workspace_from_binding(&self, _binding: &WorkspaceBinding) -> Result<WorkspaceDescriptor, WorkspaceError> {
        self.open_workspace("reopened").await
    }

    async fn create_head(&self, workspace: &WorkspaceDescriptor, request: WorkspaceHeadRequest) -> Result<WorkspaceHeadResource, WorkspaceError> {
        let head_id = WorkspaceHeadId::new();
        Ok(WorkspaceHeadResource {
            workspace: workspace.clone(),
            head: WorkspaceHeadDescriptor {
                id: head_id,
                name: request.name,
                base: request.base,
                access: request.access,
                metadata: Default::default(),
            },
            binding: WorkspaceBinding {
                provider_id: self.id(),
                workspace_id: workspace.id,
                head_id,
                access: request.access,
                payload: b"external-v1".to_vec(),
            },
            file_system: Arc::new(InMemorySessionFileStore::new()),
        })
    }

    async fn reopen_head(&self, _binding: &WorkspaceBinding) -> Result<WorkspaceHeadResource, WorkspaceError> {
        Err(WorkspaceError::NotFound)
    }

    // checkpoint, status, diff, archive, and destroy take the same &WorkspaceBinding argument
}
````

`WorkspaceProvider` and `WorkspaceProviderId` are re-exported from the facade, and a downstream crate implements the trait with no provider registry entry and no variant added to a closed backend enum. The provider names itself with an open string id through `WorkspaceProviderId::new`, which validates and returns a Result. `open_workspace(locator)` returns a descriptor with your own id and name plus free-form metadata; `open_workspace_from_binding` reopens from a persisted binding so sessions resume across restarts. `create_head` reads `request.name`, `request.base`, and `request.access`, and returns a `WorkspaceHeadResource` holding the workspace, the `WorkspaceHeadDescriptor`, the `WorkspaceBinding`, and the filesystem. The binding carries the provider id, workspace id, head id, access mode, and an opaque payload the provider defines; `InMemorySessionFileStore` backs a head for prototyping. The six post-creation methods `reopen_head`, `checkpoint`, `status`, `diff`, `archive`, and `destroy` take only the binding and return `WorkspaceError`; `NotFound` signals a head that no longer exists.

## The external-consumer fixture as a model downstream crate

````text
cargo add everruns
````

````rust
use everruns::{Agent, InMemoryEngine, LlmSimConfig, Model, ToolCall};
use external_capability_pack::{VendorSearch, math_pack};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let agent = Agent::builder()
        .instructions("Return only the answer.")
        .model(Model::simulated("4"))
        .capability("session_file_system")
        .build()?;
    let engine = InMemoryEngine::new();
    let session = engine.create(agent);
    let context = session.inspect().await?;
    assert!(context.tools.iter().any(|tool| tool.name == "read_file"));
    let turn = session.run("What is 2 + 2?").await?;
    assert!(turn.success);
    assert_eq!(turn.response, "4");

    let sim = LlmSimConfig::fixed("4").with_tool_call_sequence(vec![
        vec![ToolCall { id: "call_add".into(), name: "external_add".into(), arguments: serde_json::json!({ "a": 2, "b": 2 }) }],
        vec![],
    ]);
    let agent = Agent::builder()
        .instructions("Use external_add for exact integer sums.")
        .model(Model::simulated_with_config(sim))
        .capability("session_file_system")
        .capability(math_pack())
        .capability(VendorSearch { index: "fixtures".into() })
        .build()?;
    let session = InMemoryEngine::new().create(agent);
    let turn = session.run("What is 2 + 2?").await?;
    assert!(turn.success);
    assert_eq!(turn.tool_calls, 1, "external_add executed once");
    Ok(())
}
````

The repository's CI compiles the external-consumer fixture as an out-of-workspace application whose only framework dependency is the published `everruns` crate, with no internal core or host crate anywhere in its manifest. That surface excludes stored harness records, platform registries, backend stores, worker phases, and durable scheduling topology, and engine construction takes no config, path, or connection. The first half of `main` repeats the chapter's opening test; the second half is what the fixture adds. One polymorphic `capability()` method accepts a built-in string id, a function-returned `Definition` from the capability pack of the previous section, and an `IntoCapability` struct in the same chain.

Treat the fixture as the template for your own downstream crate. Every promoted application surface has a fixture of this kind, built on offline simulation and temporary files, and the capability, provider, builtin, event-log, workspace-provider, and execution-contract packs from the surrounding sections each have one. The application fixture is the consumer that proves the capability pack works end to end.

## everruns-host builders and HostComposition for advanced hosts

````toml
[dependencies]
everruns = "0.20.0"
everruns-core = "0.19.1"
everruns-host = { version = "0.20.5", features = ["filesystem", "web-fetch"] }
````

````rust
use std::sync::Arc;
use everruns_host::{AgentBuilder, HarnessBuilder, HostComposition, InMemorySessionFileSystemFactory, SessionBuilder};
use everruns_provider::driver_registry::DriverRegistry;
use everruns_provider::typed_id::{AgentId, HarnessId, SessionId};

let harness_id = HarnessId::new();
let agent_id = AgentId::new();
let session_id = SessionId::new();

let harness = HarnessBuilder::new("fleet-admin", INSTRUCTIONS)
    .id(harness_id)
    .capability("bashkit_shell")
    .build();
let agent = AgentBuilder::new("fleet-admin", INSTRUCTIONS)
    .id(agent_id)
    .max_iterations(12)
    .build();
let session = SessionBuilder::new(harness_id)
    .id(session_id)
    .agent(agent_id)
    .build();

let composition = HostComposition::builder()
    .driver_registry(DriverRegistry::new())
    .session_file_system_factory(Arc::new(InMemorySessionFileSystemFactory))
    .build();
````

An advanced host is a program that is itself an execution host: a server, an evaluation harness, a research runtime, or a specialized embedder that must replace storage or orchestration components. Depend on `everruns-host` only for that reason; an ordinary application never does. The framework-cli-host example pre-generates typed ids and passes them through `.id(..)` so the session can be addressed later, and a host that needs a per-session resource before the runtime exists (a `<id>.jsonl` log file, for instance) reads them back with `harness_id()`, `agent_id()`, and `session_id()`.

`HarnessBuilder::new(name, system_prompt)` seeds a portable harness definition and `AgentBuilder::new(name, system_prompt)` seeds an agent definition. `SessionBuilder::new(harness_id)` seeds a session bound to a harness. Every optional field defaults to off or empty, and the session starts in the Started state under the default organization. An empty or whitespace-only harness prompt means the harness contributes no base prompt. Builders do not validate domain invariants; the runtime does. The methods shared by the harness and agent builders come first, then the agent-only, session-only, and single-session ones:

- `capability(..)` (alias `with_capability`) attaches one capability reference; `capabilities(..)` attaches many. The session builder has the same three.
- `initial_file(InitialFile)` seeds a workspace file before any session starts.
- `network_access(NetworkAccessList)` restricts destinations at that scope.
- `parallel_tool_calls(bool)` sets the request-level preference; unset means the provider default.
- `default_model_id(ModelId)` sets that scope's default model. A session overrides it with `model_id`, giving three resolution scopes: harness default, agent default, session override.
- `metadata_entry(key, value)` and `metadata_entries(iter)` attach embedder metadata to a harness; `openrouter_attribution(http_referer, title)` writes the `openrouter.http_referer` and `openrouter.x_title` keys the OpenRouter driver sends as HTTP headers.
- Agent only: `display_name` and `description` separate from the internal `name`. Agent and session: `max_iterations(usize)`, and `tool(ToolDefinition)` plus `tools(..)`.
- Session only: `harness(id)` and `agent(id)` rebind after construction (a session may run with no agent); `title`, `goal`, `locale`, `tag`, and `tags` describe it, and session tags drive execution metadata while harness and agent definitions carry no tags; `system_prompt` overrides the prompt for that session; `organization_id(..)` and `status(..)` set a non-default organization and execution state.
- `SingleSessionBuilder`, reached through the runtime builder's `single_session(|b| ..)`, seeds one harness, one agent, and one pre-linked session in one call. Scope-named methods such as `harness_capability`, `agent_capability`, `session_capability`, `agent_initial_file`, and `session_model_id` forward to the inner builders; `harness(name, prompt)` and `agent(name, prompt)` rename or re-prompt without losing earlier configuration; `agent_plugin(name)` adds a `plugin:{name}` reference to the seeded agent.

`HostComposition` is the deployment's execution surface assembled by hand. It determines which capabilities and LLM drivers exist at runtime and which host services back them, without inheriting any product catalog. `HostComposition::default()` is empty registries plus disabled services. The builder's `capability_registry(registry)` swaps in a prepared registry and later `capability(..)` calls layer onto it, with a later registration of the same id replacing the earlier one. `driver_registry(registry)` supplies the LLM drivers, populated by provider crates' `register_driver` functions or by `register_descriptor_or_replace(DriverDescriptor::chat_only(id, factory))` for a custom chat driver. `egress_service(..)`, `utility_llm_service(..)`, and `session_file_system_factory(..)` each replace a disabled default, and `extension(Arc<T>)` injects a type-keyed service that tools resolve from their context. `everruns_host::runtime_capability_registry()` returns a registry populated with the integrations selected by the host's features: `session_file_system` and `web_fetch` are present, and `bashkit_shell` is absent when `bashkit` is off. `compose_runtime_capability_registry(registry)` layers the same integrations onto a registry you own while preserving your entries, filling in only a missing session or session-storage capability. `runtime_egress_service()` returns the host's runtime egress service.

On a built composition, `register_capability(arc)` makes a capability discovered mid-session resolvable without rebuilding and rejects duplicate ids and alias collisions; `register_capability_overriding` replaces by canonical id and belongs at composition time, not after deployment where a silent override would hide a bug. `is_capability_registered(id)` checks canonical ids and aliases, but registration is not activation: `activate_capability` still decides per-session enablement. `capability_registry()` returns an immutable snapshot, so readers never see a half-built registry, and a capability registered on a clone is not visible to the original. With the `observability` feature you can attach an `OtelEventListener` selecting `TraceConventions` (an OpenInference set is available) or a `BraintrustListener` configured by `BraintrustConfig`, and a `CompositeEventListener` fans out to several backends at once. `init_telemetry(TelemetryConfig)` initializes process-wide telemetry in one call.

## In-process runtime operations

````toml
[dependencies]
everruns-host = { path = "../../crates/host", features = ["bashkit", "filesystem"] }
everruns-provider = { path = "../../crates/provider" }
everruns-integrations-bashkit = { path = "../../integrations/bashkit" }
everruns-llmsim = { path = "../../crates/drivers/llmsim", features = ["host"] }
````

````rust
use everruns_host::InProcessRuntimeBuilder;
use everruns_integrations_bashkit::BashkitShellCapability;
use everruns_llmsim::{LlmSimConfig, LlmSimRuntimeExt};
use everruns_provider::tool_types::ToolCall;
use serde_json::json;

let runtime = InProcessRuntimeBuilder::new()
    .host_composition(composition)
    .capability(BashkitShellCapability)
    .harness(harness)
    .agent(agent)
    .session(session)
    .llm_sim_as_default(LlmSimConfig::fixed("The api service runs 2 replicas.").with_tool_call_sequence(vec![
        vec![ToolCall { id: "call_help".into(), name: "bash".into(), arguments: json!({ "commands": "everruns --help" }) }],
        vec![ToolCall { id: "call_list".into(), name: "bash".into(), arguments: json!({ "commands": "everruns fleet list" }) }],
        vec![],
    ]))
    .build()
    .await?;

let result = runtime.run_text_turn(session_id, "How many replicas does api run?").await?;
assert!(result.success);
assert_eq!(result.tool_calls_count, 2);
````

````text
cargo run -p everruns-framework-cli-host -- --offline
````

Behind the framework-cli-host example's `--offline` flag, the run is deterministic, with no key and no network, and the simulator issues the shell calls. `InProcessRuntimeBuilder::new()` starts pre-loaded with runtime-safe built-in capabilities, an empty driver registry so you choose the provider explicitly, the runtime egress service, and an in-memory session filesystem factory. `build().await` returns an `InProcessRuntime` that executes turns inside your process without the durable engine or the control-plane server; it is cheaply cloneable and shared across tasks. Registering the stock `BashkitShellCapability` from `everruns-integrations-bashkit` on the runtime and granting `bashkit_shell` to the harness by name gives the model a `bash` tool with a `commands` argument.

The simulator attaches through neutral provider seams, because `everruns-host` contains no simulation code. `provider(..)` registers a canonical provider and rejects duplicate identities at build. `replace_provider(..)` swaps in a replacement under an existing id without changing model selection. `provider_with_default_model(provider, model_id)` registers a provider and makes its named model the default in one step. `default_model(ModelSpec)` (alias `model_spec`) selects the default credential-free model directly. The `host` feature of `everruns-llmsim` adds the `LlmSimRuntimeExt` extension trait with two methods over those seams: `.llm_sim(config)` calls `replace_provider(llm_sim_provider(config))` and leaves model selection alone, so pair it with `default_model(..)` if the simulator should serve a selected model; `.llm_sim_as_default(config)` calls `provider_with_default_model(llm_sim_provider(config), LLMSIM_MODEL_ID)`, which is the compact test setup. Build fails fast with a descriptive configuration error when no default model has been configured, and the message names `model_spec(..)` and `provider_with_default_model(..)` as remedies, along with the simulator's `.llm_sim_as_default(..)`.

Other builder knobs: `host_composition(composition)` replaces the entire platform composition; `capability(..)` and `driver_registry(..)` mirror the composition builder; `harness(..)`, `agent(..)`, and `session(..)` seed definitions into the runtime stores; `with_plugin_dir(path)` loads a plugin from a directory with compile errors surfaced before build. `session_file_system_factory_context(..)` supplies host dependencies to the filesystem factory. `provider_retry_config(..)` overrides the bounded provider retry policy, and `provider_stall_timeout(duration)` overrides the timeout for a provider stream that produces no output. The host's tool surface extends through `with_tool_augmentor(..)`, and per-tool context comes from `with_tool_context_extensions_factory(..)`, whose factory receives the org and session on each call. Subagent delegation plugs in through `with_subagent_delegate_factory(..)`.

Operations on the built runtime:

- `run_turn(session_id, input)` executes one turn from any input message and `run_text_turn(session_id, text)` from plain text; the input is recorded as the canonical `input.message` event. The `TurnResult` carries `response`, `iterations`, `tool_calls_count`, `success`, `error`, `stop_reason`, and `turn_id`. A failed turn reports an empty response and zero tool calls alongside the error and stop reason.
- `run_steerable_turn(session_id, input, turn_id, steering)` takes a caller-supplied turn id and a `TurnSteering` handle; a session has at most one active turn, and steering is how further input joins it. `TurnSteering::try_push(input)` from another task reports atomically whether the message joined the current turn or was rejected as `Closed` or `Full` (capacity 256), returning the input for requeue, and that receipt is authoritative. Too-late input is persisted as the start of the next turn and still passes the user-prompt-submit hooks.
- `activate_capability(session_id, ref)` activates a registered capability on one live session after validation and dependency resolution; re-activation reports no change. `deactivate_capability(session_id, id)` removes a session-scoped capability, and inherited agent or harness capabilities cannot be removed at the session layer. Each returns a `CapabilityDelta` saying whether it changed, whether it is active, and whether prompt, tool, hook, command, or MCP surfaces must be refreshed.
- `messages(session_id)` reads the full message history reconstructed from the event log. `load_context(session_id)` assembles the full turn context without executing a turn. `events()` replays all durable events and requires exactly one seeded session.
- `execute_command(session_id, request)` runs a slash command such as `/model` declared by any capability in the session's resolved chain, and `list_commands(session_id)` lists them.
- `plugin_warnings()` exposes non-fatal plugin compile warnings and `plugin_capability(name)` returns the hydrated reference to seed onto a harness or agent. Bare plugin references with empty configs are hydrated automatically; explicit non-empty configs are left untouched.
- `in_process_internal_org_id(public_org_id)` maps a public organization id to the internal id the runtime uses for per-org stores.

## Host adapters, execution drivers, and turn-context assembly

````rust
use async_trait::async_trait;
use everruns_host::{ResolvedTurnInputs, RuntimeHostAdapter};

#[async_trait]
impl RuntimeHostAdapter for MyHost {
    async fn load_resolved_turn(&self, org_id: i64, session_id: SessionId) -> Result<ResolvedTurnInputs> {
        // one batched load: snapshot, messages, MCP tool definitions
    }

    async fn set_session_status(&self, org_id: i64, session_id: SessionId, status: SessionExecutionState) -> Result<()> {
        // persist Active, Idle, or WaitingForToolResults
    }

    fn capability_registry(&self) -> CapabilityRegistry { self.capabilities.clone() }
    fn driver_registry(&self) -> DriverRegistry { self.drivers.clone() }
}
````

When the in-process runtime is not enough, you implement `RuntimeHostAdapter` and reuse the shared input, reason, and act orchestration. Your crate supplies persistence, session lifecycle plumbing, event delivery, and its own orchestration backend. Four methods are required. `load_resolved_turn(org_id, session_id)` returns the turn's inputs in one batched load, and missing or inactive records fail before execution. `set_session_status(..)` persists lifecycle state. `capability_registry()` and `driver_registry()` return the registries. Every other hook defaults to none and opts a service in: provider credentials, a utility LLM, outbound egress, session storage, user connections, image and file resolvers, tool context extensions, a subagent delegate, a tool augmentor, leased resources, session resources and tasks, budget and payment authorities, session creation authority, an outbound tool rate limiter, a subagent spawn store, a stream heartbeater, and turn cancellation.

The phase functions run each phase against your adapter. `execute_input_activity` marks the session active and emits `session.activated` and `turn.started`. `execute_reason_activity` runs prompt assembly and tool exposure, then the LLM call; `execute_reason_activity_with_prompt_messages` takes the ids of synthetic user messages a host injects between iterations so they cross the same policy boundary as the original input. `execute_act_activity` requires an org id and validates tool context services up front, so host wiring errors surface as configuration failures instead of late tool-call failures. Before a turn, `detect_dependency_blocker` detects an archived or deleted harness or agent and surfaces a user-facing dependency error. `RuntimeSessionLifecycle::new(adapter, org_id, session_id)` drives lifecycle events and status through `turn_started`, `turn_completed`, `emit_turn_completed`, `emit_session_idled`, `turn_sealed`, `turn_failed`, `turn_failed_with_disclosure`, `user_prompt_blocked`, and `waiting_for_tool_results`.

Beneath the adapter sits the `Execution` trait from `everruns-engine`, the documented boundary between the engine and a host. The host crate implements it with process-local state and the durable crate with checkpointed state between scheduled activities; the contract is synchronous and sans I/O. The kernel exposes `InputAtom`, `ReasonAtom`, and `ActAtom` step values plus phase values, and it has no dependency on host, platform, server, worker, or durable crates. `InputAtom::new(retriever)` accepts your own `MessageRetriever` implementation, and `ExecutionContext::new(SessionId::new(), TurnId::new(), MessageId::new())` constructs the input to engine atoms with a generated execution id.

````rust
use everruns_host::{StoreTurnContextResolver, inspect_turn_context};

let resolver = StoreTurnContextResolver::new(
    harness_store, agent_store, session_store, message_retriever, provider_store,
    capability_registry, driver_registry,
)
.with_file_store(file_store)
.with_session_storage(session_storage);
````

Turn-context assembly is the other seam a host controls. `StoreTurnContextResolver::new(..)` builds a resolver from your own stores plus the two registries; `with_file_store(..)` lets dynamic prompt capabilities read session files and `with_session_storage(..)` lets them read persisted session state. The free function `assemble_turn_context(..)` assembles directly from stores without a resolver object, and `assemble_turn_context_from_snapshot(..)` skips the store round-trip when the host already loaded the snapshot. `inspect_turn_context(..)` allows empty history so a debugging or preview UI can show what the model would see before any message exists. When your host cannot preassemble a context before a reason atom, implement `TurnContextResolver` with its single `resolve_turn_context(request)` method; the request carries the session id, harness id, optional agent id, and MCP tool definitions.

The core types underneath are credential-safe by construction: credentials, tenant records, and host lifecycle entities never leak through model identity or diagnostic values. `ResolvedModelExecution` carries only the model name, provider key, driver kind, and an opaque driver, and credentials never appear in its Debug output. `ResolvedTurnContextInput` supplies pre-filtered history and an execution snapshot while store access and filtering stay in the host. A reason step reads everything from one `AssembledTurnContext`. `assemble_resolved_turn_context(input, registry, file_store, session_storage)` builds it, scoping a supplied filesystem to the workspace before capabilities see it and taking the locale from the last user message's controls or the snapshot. `resolve_snapshot_capabilities(snapshot, registry)` reconstructs the effective overlay with scopes folded as harness, then agent, then session. Setting `snapshot.blueprint_id` runs the session against a registered blueprint instead of the overlay, with a default of 20 max turns. Two typed errors cover assembly failures: `NoMessages` when a session has no model-visible messages and messages are required, and `ModelNotConfigured` (non-retryable) when no model resolves. Your own implementations report configuration problems with `AgentLoopError::config(msg)`. `InMemoryProviderStore` with `with_default(..)` and `add_model(..)` tests model resolution in process.

## Architecture, crate layering, and design commitments

A real application wires three layers as independent dependencies: the facade, one driver crate for the model provider, and one or more integration crates for tools. The research-agent example's manifest lists exactly `everruns`, `everruns-openrouter`, and `everruns-integrations-brave-search` as separate dependencies plus tokio, with the facade's `web-fetch` feature enabled, and no umbrella or meta crate sits between them. Application and host setup paths converge before provider resolution and engine execution, which is why facade behavior matches the hosted worker and server. Which crate you depend on follows from what you are building:

- Applications depend on `everruns`. The default facade graph contains no `everruns-platform`, Reqwest, Rustls, or Hyper.
- Advanced hosts add `everruns-host` and focused siblings. It is the only low-level host boundary; there is no separate runtime crate, and such a host should use low-level extension traits rather than expect every backend re-exported through one facade.
- Capability packs depend on `everruns-capability`, the neutral contract crate.
- Provider packs depend on `everruns-provider`, the provider SPI with typed ids and `AgentLoopError`.
- Kernel embedders depend on `everruns-engine`, the shared Input/Reason/Act turn engine. It depends only on core, capability, and provider, and a dependency-direction test enforces that it never depends on host, server, worker, platform, durable, or scale crates.
- Simulation depends on `everruns-llmsim`; test helpers on `everruns-test-support`. Core carries neither dependency nor any feature for them; `#[cfg(test)]` fixtures are its only gate.
- `everruns-core` is the shared contract crate every part of the ecosystem speaks. It models agents, harnesses, sessions, messages, and events with one set of types, plus capability and tool traits, provider-neutral execution inputs, and context assembly for the input, reason, act flow. It is storage-agnostic: execution is expressed through focused traits such as `MessageRetriever`, `ToolExecutor`, `EventEmitter`, and `ProviderStore`, and a host decides whether memory, PostgreSQL, gRPC, or something else backs them, just as it can replace the high-level work queue that scopes background tasks and direct wakes to a session, whose default provider is offline and process-local. Environment-backed integrations (filesystem, Bashkit, web fetch, Lua, MCP, HTTP transport) never leak into it; hosts select those edges explicitly.
- The `local` crate supplies the opt-in local profile that combines real workspace files with local task and schedule state; local persistence never serializes agent behavior, since an agent may contain process-local drivers, handlers, and closures, and after a restart the application reconstructs the agent in code and attaches it to a new engine before resuming, with the id verified against the local session catalog. `macros` implements the tool attribute the facade re-exports, and hosted product capabilities sit in `everruns-platform`.

````rust
use std::time::Duration;
use everruns_core::config::{env_bool, env_duration_secs, env_or, env_string};

let port = env_or("EVERRUNS_CONFIG_DOC_TEST_PORT", 8080_u16);
let bind = env_string("EVERRUNS_CONFIG_DOC_TEST_BIND_ADDR", "127.0.0.1");
let enabled = env_bool("EVERRUNS_CONFIG_DOC_TEST_FEATURE_ENABLED", false);
let timeout = env_duration_secs("EVERRUNS_CONFIG_DOC_TEST_TIMEOUT_SECS", Duration::from_secs(30));
````

Core also ships the shared environment-variable helpers in `everruns_core::config`, so services and workers parse configuration the same way. The default value's type drives the parse target, `env_duration_secs` reads a whole-seconds count into a `Duration`, and `ConfigError` covers the fallible loading paths.

## Examples catalog and root walkthroughs

````text
cargo run -p everruns --example engine_sessions
cargo run -p everruns --features local --example session_history
cargo run -p everruns --features openai --example github_monitor -- --simulate
````

Start with the public `docs/framework/` section, the canonical usage guide, which begins offline before requiring any provider. The maintained public catalog is in `crates/everruns/examples`; every program imports only the facade, is compiled in CI, and can be copied into an application unchanged. Eight of them are fully offline and need no keys: `capability_configuration`, `canonical_events`, `engine_sessions`, `live_session`, `session_work`, `workspace_heads`, `workspace_policy`, and `session_history`. None uses `--features openai`; `session_history` and `workspace_heads` need `--features local`. Live-provider crate examples use `--features openai`, default to `gpt-5.6-terra`, and require `OPENAI_API_KEY`; `github_monitor` also offers an offline simulation mode behind `--simulate`. Copy exact commands from the `examples/README.md` beside the sources. Examples that demonstrate low-level host internals are advanced-host examples, and importable hosted Platform agent definitions are kept separately in `examples/agents`.

The root `examples/` folder holds complete end-to-end walkthroughs, each shipping the program, instructions, fixtures where applicable, and recording scripts. Run them from a repository checkout, because their manifests point at workspace crates. `cargo run` makes a real provider call.

- Support Agent (OpenAI `gpt-5.6-terra`) chooses between MFA recovery, lockout, and browser troubleshooting from facts and policy; the recommended starting point for typed tools. `export OPENAI_API_KEY="your-key" && cargo run -p everruns-support-agent`
- Everruns Support Agent (Anthropic `claude-opus-5`) searches and reads citable official documentation snapshots. `export ANTHROPIC_API_KEY="your-key" && cargo run -p everruns-framework-support-agent`
- Coding Review Agent (Anthropic `claude-sonnet-5`) reads a refund contract and executes a fixed regression before reporting a defect; the recommended starting point for restricted execution. `export ANTHROPIC_API_KEY="your-key" && cargo run -p everruns-coding-review-agent`
- Research Agent (OpenRouter `z-ai/glm-5.2`) searches and fetches primary sources before writing a cited brief; the recommended starting point for reusable capabilities. `export OPENROUTER_API_KEY="your-key" && export BRAVE_SEARCH_API_KEY="your-key" && cargo run -p everruns-research-agent`
- Incident Commander Agent (Meta Model API `muse-spark-1.3`) investigates fixture telemetry and persists an evidence-backed incident update. `export MODEL_API_KEY="your-key" && cargo run -p everruns-incident-commander-agent`
- Bashkit Repo Agent (OpenAI `gpt-5.6-terra`) cuts a release in a real repository with the sandboxed Bashkit shell as its only tool, then verifies the result on disk. `export OPENAI_API_KEY="your-key" && cargo run -p everruns-bashkit-repo-agent`

````text
cargo run -p everruns-support-agent -- "cust_locked reset their password but cannot sign in. What should they do?"
cargo run -p everruns-bashkit-repo-agent -- --interactive /tmp/release-run
cargo test -p everruns-bashkit-repo-agent
bash examples/bashkit-repo-agent/demo/record.sh --check
````

Pass a question as positional arguments to override the default prompt. Pass `--interactive` to type the task at a prompt (bashkit-repo, everruns-support, and support accept it). The bashkit-repo agent accepts a positional scratch directory so the mutated repository survives for inspection; its default workspace is temporary and removed on exit. Missing credentials, provider errors, unsuccessful turns, and failed disk assertions exit nonzero. Every walkthrough prints through one shared `demo-support` crate (dependency `everruns-example-demo`, not needed in production) so each package contains only its agent: `demo::run(&session, &request)` runs the session and streams events to the terminal, and `demo::banner`, `demo::field`, `demo::section`, `demo::check`, and `demo::body` print labeled output. The bashkit-repo walkthrough re-reads the workspace after the run through `fixture::verify(root, release_date)` and fails the process if any release invariant is false, so a confident model answer cannot make a broken release pass.

`cargo test -p <example-package>` runs each example's offline suite without credentials, and tool helpers are unit-tested with plain `#[tokio::test]` functions that construct no engine, agent, or provider. Recording scripts accept `--check`. CI runs these offline checks and does not grade model output. Each walkthrough ships a GIF and a transcript (`demo/demo.gif` and `demo/transcript.txt`, or `src/demo.gif` and `src/demo.txt`). Re-record with the `record.sh` scripts, which need VHS, ffmpeg, a VHS-compatible browser, funded credentials, and for some examples Python 3; credentials come from the exported key or Doppler project `everruns-dev`, config `dev`. Replay an existing transcript into a GIF without a model call with `(cd src && python3 render_demo.py && vhs demo.tape)`.

Source layout is consistent: `src/main.rs` holds the flow, `src/agent.rs` the agent, `src/tools.rs` the bounded tools, `src/fixture.rs` or fixture text files the starting state, `src/resources/` the prompt and sample data, and `demo/` or `src/record.sh` with `src/render_demo.py` the recording. To adapt an example, replace the fixture, acceptance criteria, and verifier together. The hackernews-reader example is the other extreme: a complete agent in one Markdown file, with YAML frontmatter (`name`, `description`, `tags`, `capabilities`) followed by plain Markdown instructions and no special DSL.

## Feature flag reference

Every crate in this chapter ships with `default = []` except the facade, whose defaults are covered in the installation chapter.

`everruns-llmsim`:

- `default = []`. With no features the crate is a plain provider built on `everruns-provider`; `everruns-host` is an optional dependency.
- `host = ["dep:everruns-host"]`. Pulls in `everruns-host` and enables the `LlmSimRuntimeExt` extension trait with `.llm_sim(config)` (registration only) and `.llm_sim_as_default(config)` (registration plus default model selection) on `InProcessRuntimeBuilder`. Enabled by the host crate's dev-dependency (`everruns-llmsim = { path = "../drivers/llmsim", features = ["host"] }`) and by the framework-cli-host example's dependency (`everruns-llmsim = { path = "../../crates/drivers/llmsim", features = ["host"] }`).

`everruns-host`:

- `default = []`. Zero-footprint host with no optional tools, network egress, or integrations compiled in; opt in feature by feature.
- `direct-egress = ["dep:reqwest"]`. Pulls in `reqwest` with the `json`, `stream`, `rustls`, and `blocking` features and exposes `DirectEgressService` for direct outbound network calls. Implied by `mcp`, `bashkit`, `web-fetch`, `duckduckgo`, `lua`, and `observability`.
- `observability = ["direct-egress", "tokio/time", "everruns-provider/tls-aws-lc-rs", "dep:rand", "dep:opentelemetry", "dep:opentelemetry_sdk", "dep:opentelemetry-otlp", "dep:tracing-opentelemetry", "dep:tracing-subscriber"]`. Pulls in direct egress, tokio timers, TLS via aws-lc-rs, the OpenTelemetry SDK and OTLP exporter, and the tracing bridges; exposes the `observability` module (OTel, OpenInference, Braintrust, composite listeners, `init_telemetry`). The host's dev-dependency on `opentelemetry_sdk` uses `features = ["testing"]` for the listener tests.
- `utility-openai`. Exposes `OpenAiUtilityLlmService`, `SystemUtilityLlmConfig`, and the `UTILITY_OPENAI_API_KEY_ENV` variable name so OpenAI can serve as the system utility LLM for background tasks such as summarization or titling.
- `filesystem`. Compiles in the session filesystem integration; used in the documented advanced-host manifests (`features = ["filesystem", "web-fetch"]` and `features = ["bashkit", "filesystem"]`).
- `web-fetch`. Compiles in the web fetch integration (`web_fetch` capability present in the runtime registry); implies `direct-egress`.
- `bashkit`. Compiles in the sandboxed shell integration; `bashkit_shell` is absent from the runtime registry when it is off; implies `filesystem` and `direct-egress`.
- `mcp`, `duckduckgo`, `lua`. Compile in the MCP client, DuckDuckGo search, and Lua integrations respectively; each implies `direct-egress`. Their behavior is covered in the capability boundaries chapter.
- `--no-default-features` on `everruns-host` yields a dependency graph with no `everruns-platform`, Reqwest, Rustls, or Hyper.

`everruns-core`:

- `default = []`. Intentionally empty so the minimal footprint is the baseline.
- `openapi = ["dep:utoipa", "everruns-provider/openapi"]`. OpenAPI schema generation, enabled by the server.
- `tree-sitter-outlines = ["dep:tree-sitter", "dep:tree-sitter-rust", "dep:tree-sitter-typescript", "dep:tree-sitter-python"]`. Structural outlines for Rust, TypeScript, and Python.

`everruns` (facade), as used by this chapter's examples:

- `openai`. Required by the live-provider crate examples (`cargo run -p everruns --features openai --example ...`); enables the OpenAI driver.
- `local`. Required by the `session_history` and `workspace_heads` examples; enables local persistence.

`everruns-test-support`:

- `fixtures`. Enables the writable fixtures used by the facade's acceptance tests (`everruns-test-support` with `features = ["fixtures"]`).

`everruns-provider` (transitively):

- `tls-aws-lc-rs` is pulled in by the host's `observability` feature; `openapi` is pulled in by core's `openapi` feature.

`tokio` (dependency and dev-dependency of the framework-cli-host example):

- `features = ["macros", "rt-multi-thread"]`.

Runtime switches that are not Cargo features: an integration plugin submitted at link time may name a `feature_flag` resolved from a `FEATURE_<NAME>` environment variable, fail-closed, and `LLMSIM_DEMO=guarded|tasks|monitor` selects a prebaked simulator script.

*2026-09-17 03:24 - claude-fable-5.1*
