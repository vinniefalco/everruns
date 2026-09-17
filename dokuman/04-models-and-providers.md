<!-- source: everruns/everruns @ 6cf2c15e5 (crate everruns 0.20.0). The working tree fast-forwarded to 7d9e07a0b (0.21.1) during extraction; identifiers and examples were audited against 6cf2c15e5. -->
# Models and Providers

An agent is only as useful as the model behind it, and the model you start with is rarely the model you ship. In `everruns` the choice is two builder calls: `.model(..)` names the model and `.provider(..)` names the service that serves it, and both are application configuration rather than code. What you hand to `.provider(..)` ranges from nothing at all, for the offline simulator, to the `OpenAI` config value behind the `openai` Cargo feature, a vendor driver crate's ready-made provider, or a `Provider` you assemble yourself around any `ChatDriver`.

## Choosing a model and provider: `Model`, plain model ids, the offline simulator, and OpenAI through the `openai` feature

### The simulator and the plain model id

The simplest agent runs against no provider at all.

````rust
use everruns::{Agent, BuildError, Model};

fn build() -> Result<Agent, BuildError> {
    Agent::builder()
        .instructions("Answer deterministically.")
        .model(Model::simulated("fixed response"))
        .build()
}
````

`Model::simulated` is backed by the in-process simulator crate `everruns-llmsim`, registered under the provider id `llmsim` with the model id `llmsim-model`. It always replies with the string you gave it, needs no key and no network, and is production-safe rather than a test-only shim. Nearly every example and test in the framework uses it, which is why `cargo run -p everruns --example canonical_events` works with no feature flags. A second configuration echoes the latest user input instead of returning a constant, and can add a response delay.

````rust
use everruns::{LlmSimConfig, Model};
use std::time::Duration;

let model = Model::simulated_with_config(
    LlmSimConfig::echo().with_response_delay(Duration::from_millis(100)),
);
````

The `live_session` example uses this form and prints "Offline simulator: echoes the latest user input; no model inference." when it runs offline.

A `Model` value bundles a model id with the provider used to reach it, through `Model::new(id, provider)`. Bundling suits applications where model selection is configuration. The other path passes a plain `&str` or `String` to `.model(..)` and attaches the provider separately with `.provider(..)`. The string is credential-free and names the provider-visible model; when the agent builds, the framework resolves it against the attached provider. Examples pin the id as a constant such as `pub const MODEL: &str = "gpt-5.6-terra";` and call `.model(MODEL)`. You never construct an execution-facing model spec yourself.

An agent accepts exactly one provider. A live string model with no provider fails `build()` with `BuildError::MissingProvider`; two providers fail with `BuildError::MultipleProviders { registered }`, where `registered` lists the provider keys in sorted order. Both are construction-time errors. A session can override the agent's default model, which chapter 05 covers.

### OpenAI through the `openai` feature

The default build compiles no network provider code. Enabling the `openai` feature on the `everruns` crate adds the `OpenAI` config value and its `OpenAIError` type, both also exported from the prelude, and today `openai` is the only provider feature the framework crate exposes. Every other vendor is a separate driver crate that the application depends on directly.

````rust
use everruns::{Agent, InMemoryEngine, OpenAI};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let agent = Agent::builder()
        .instructions("You are concise.")
        .provider(OpenAI::from_env()?)
        .model("gpt-5-mini")
        .build()?;

    let result = InMemoryEngine::new()
        .create(agent)
        .run("Write one sentence about reliable agents.")
        .await?;
    println!("{}", result.response);
    Ok(())
}
````

`OpenAI::from_env()` reads `OPENAI_API_KEY` (required) and `OPENAI_BASE_URL` (optional). It is the sanctioned entry point for standalone programs and dev loops. Run it with `export OPENAI_API_KEY=sk-...` first, or inline: `OPENAI_API_KEY=sk-... cargo run -p everruns --features openai --example hello`. A missing or empty key is `OpenAIError::MissingEnvVar { var }`, whose display text is "required environment variable OPENAI_API_KEY is not set". The error names the variable and never the value, and an empty string counts as missing.

When your application manages configuration itself, use `OpenAI::new(api_key)`, which reads nothing from the environment. The common pattern reads the variable in `main`, fails early with `?`, and passes the key in.

````rust
let api_key = std::env::var("OPENAI_API_KEY")?;
let agent = Agent::builder()
    .instructions("You are concise.")
    .provider(OpenAI::new(api_key))
    .model("gpt-5.6-terra")
    .build()?;
````

`OpenAI::base_url(..)` is chainable on either constructor and points the provider at an OpenAI-compatible proxy or a self-hosted endpoint: `OpenAI::new("sk-explicit").base_url("https://custom.example/v1")`. The framework crate sets no default of its own; when unset, the driver default applies. Debug-printing an `OpenAI` value shows the base URL and renders the key as `[REDACTED]`, and converting it into a provider yields the provider id `openai` without leaking the key. When the config value is not enough, `everruns::providers::openai` re-exports the low-level `OpenAIChatDriver`, `OpenAICompletionsChatDriver`, and `register_driver` from the driver crate.

The `live_session` example uses both paths. It defaults to the simulator, and when built with `--features openai` and run with `-- --live` it switches to `OpenAI::from_env()?` with `.model("gpt-6-astra")` under a `#[cfg(feature = "openai")]` block.

## The `everruns-openai` driver crate: Responses and Chat Completions drivers, Azure OpenAI, model discovery, embeddings

### Assembling a provider

The `OpenAI` config value from the previous section is a thin wrapper around the `everruns-openai` crate, which you can use directly when you need more than a key and a base URL. It implements the neutral `ChatDriver` contract from `everruns-provider` and maps provider-neutral messages, tools, and reasoning onto the OpenAI wire format. The crate ships one function per deployment shape, each returning a ready-to-use `Provider` value.

````rust
use everruns_openai::{azure_provider, completions_provider, provider};

let api_key = std::env::var("OPENAI_API_KEY")?;
let azure_key = std::env::var("AZURE_OPENAI_API_KEY")?; // your application's own variable

// Hosted OpenAI over the Responses API. Bearer auth, base URL https://api.openai.com/v1.
let openai = provider("openai", api_key.clone());

// Your Azure OpenAI resource. The key is sent as an `api-key` header.
let azure = azure_provider("azure", "https://my-resource.openai.azure.com/openai/v1", azure_key);

// Hosted OpenAI over the legacy Chat Completions API.
let legacy = completions_provider("openai-legacy", api_key);
````

The first argument is the provider id, a label you choose. `provider` wires `OpenAIChatDriver`, the Responses API driver, with `BearerAuth` and the default base URL `https://api.openai.com/v1`, so `endpoint().url("responses")` resolves to `https://api.openai.com/v1/responses`. `completions_provider` wires `OpenAICompletionsChatDriver` to the same host and resolves `chat/completions` instead. `azure_provider` takes your resource endpoint as the base URL, sends the key as an `api-key` header rather than a Bearer token, and uses the base path `/openai/v1`. Use it rather than a generic base URL override for Azure, because the dedicated `azure_openai` provider type resolves endpoint and model behavior correctly and is recognized as a stateful host. Neither driver crate reads an environment variable for you; the application reads its own variables, as above, and passes the key in.

The Responses driver is the recommended default and performs better with reasoning models; the Chat Completions driver exists for `/v1/chat/completions` gateways and older integrations, under the driver id `openai_completions`. Both deliver streaming text, tool calls, and reasoning as neutral types, and any of these providers accepts `.base_url(url)` to target a proxy, mock, or gateway.

Hosts that create drivers from configuration register the crate once into a `DriverRegistry`, the lookup table of driver factories.

````rust
use everruns_openai::register_driver;
use everruns_provider::DriverRegistry;

let mut registry = DriverRegistry::new();
register_driver(&mut registry);
````

One call registers all three driver ids, `openai`, `azure_openai`, and `openai_completions`. Identical messages and config produce identical results across them. OpenRouter is registered separately by its own crate.

### What the Responses driver turns on

`OpenAIChatDriver::new()` enables stateful responses, native features, and prompt-cache options with no further configuration. Stateful continuation and server-side context compaction are tied to the recognized hosts, `api.openai.com` and Azure OpenAI. The Responses and Azure drivers report `supports_stateful_responses()` as true and the Chat Completions driver as false, which the engine uses to preserve tool results whose originating calls exist only in prior server-side state.

You never supply a cache key; the driver derives a deterministic one from stable request inputs, so OpenAI prompt caching happens on its own. Compaction, by contrast, is something you ask for. `compact_conversation(endpoint, request)` calls `/v1/responses/compact`, keeps user messages verbatim, and replaces assistant messages, tool calls, and tool results with an encrypted compaction item; check `can_compact()` first, since OpenAI's Responses API supports it and a custom endpoint may not.

When the provider is created, and again on each sync, `list_models(endpoint)` derives the `/models` URL from the configured base URL and returns `DiscoveredModel` records tagged `chat` or `embeddings`. Ids starting `gpt-`, `o1`, `o3`, `o4`, or `chatgpt-` count as chat and `text-embedding-` as embeddings; image, TTS, whisper, moderation, realtime, sora, and codex models are dropped. Discovery is restricted by host: only `api.openai.com` and Azure hosts (`*.openai.azure.com`, `*.services.ai.azure.com`) are probed, and a custom proxy returns `None` so credentials never reach look-alike infrastructure.

### Embeddings

The `openai` driver id declares three services: chat, realtime, and embeddings. `OpenAIEmbeddingsDriver` posts to `/v1/embeddings` and is created from the registry with `create_embeddings_driver(&ProviderConfig)`. The registry never reads the environment, so the key goes in through `ProviderConfig::with_api_key`; the registry-built driver is bound to the provider's endpoint, which defaults to `https://api.openai.com/v1`.

````rust
use everruns_provider::{DriverId, EmbedRequest, ProviderConfig, ProviderEndpoint};

let api_key = std::env::var("OPENAI_API_KEY")?;
let driver = registry.create_embeddings_driver(
    &ProviderConfig::new(DriverId::OpenAI).with_api_key(api_key),
)?;

let response = driver
    .embed(&ProviderEndpoint::default(), EmbedRequest {
        model: "text-embedding-3-small".into(),
        texts: vec!["first".into(), "second".into()],
    })
    .await?;
````

One request embeds the whole batch and returns `EmbedResponse.embeddings` as `Vec<Vec<f32>>` in input order, even when the API answers out of order. `usage_tokens` reports total tokens and `actual_cost_usd` holds a real dollar cost when an OpenAI-compatible gateway such as OpenRouter reports one; direct OpenAI leaves it `None`. The driver works against any OpenAI-compatible base URL, and a provider with no base URL fails with `EmbeddingsDriverError::Provider("provider has no base URL")`. Errors split into `Transport` and `Provider`, and 4xx other than 429 fails fast rather than consuming the retry budget. Azure OpenAI and Chat Completions advertise chat only, so requesting an embeddings driver from them fails cleanly.

## The OpenAI Chat Completions protocol driver and GPT-6 Astra request validation

### One driver for every OpenAI-compatible endpoint

Beneath `OpenAICompletionsChatDriver` sits `OpenAIProtocolChatDriver`, the base Chat Completions implementation in `everruns-provider`. It talks to any OpenAI-compatible chat endpoint, hosted or local, and it knows nothing about where it is pointed: endpoint and auth come from the `Provider` it is attached to. It is available behind the neutral crate's default `http` feature.

````rust
use everruns_provider::{BearerAuth, LlmRetryConfig, OpenAIProtocolChatDriver, Provider};

// The driver reads no environment; your application supplies the key.
let api_key = std::env::var("GROQ_API_KEY")?;

let driver = OpenAIProtocolChatDriver::new().with_retry_config(LlmRetryConfig::aggressive());
let provider = Provider::new("groq", driver)
    .base_url(base_url)
    .auth(BearerAuth::new(api_key));
````

The driver streams text as `LlmStreamEvent::TextDelta` events, and it also offers a native non-streaming path that sends `stream: false` and returns one complete `LlmResponse`. All four conversation roles map onto the wire, with tool results linked to their originating call by `tool_call_id`. `LlmContentPart::Image { url }` becomes an `image_url` part, audio becomes `input_audio` in `wav` format, and a base64 PDF data URL with a filename becomes a `file` part. Your typed tools are sent as `function` tools and come back as structured `ToolCall { id, name, arguments }` values with parsed JSON arguments. `with_retry_config` sets retry aggressiveness per driver; `LlmRetryConfig { max_retries: 0, ..Default::default() }` disables retries.

Reasoning models served over Chat Completions (DeepSeek-R1, Qwen, Groq, Fireworks) stream their chain-of-thought as separate `LlmStreamEvent::ReasoningDelta` events, and the accumulated reasoning persists as a durable artifact. Without any configuration, the driver also drops tool-result messages whose originating tool call is no longer visible before sending, so trimming or compacting history never produces a permanent 400, and it discards pending tool calls when the finish reason is `length` or `content_filter`, so a truncated turn never executes a half-formed call.

### Per-call request options

Every option in this section is a field on `LlmCallConfig`, the per-call configuration. Build one from a model id with `LlmCallConfig::new(model)` or through `LlmCallConfigBuilder`.

````rust
let config = LlmCallConfigBuilder::from_config(LlmCallConfig::new("gpt-5.6-terra"))
    .reasoning_effort(ReasoningEffort::High)
    .speed("flex")
    .verbosity("high")
    .max_tokens(1024)
    .with_metadata("session_id", "s1")
    .build();
````

`temperature` and `max_tokens` tune sampling and the output cap; the Responses API maps `max_tokens` to `max_output_tokens`. `reasoning_effort` takes one of a fixed set of levels: None, Minimal, Low, Medium, High, Xhigh, Max. An explicit `ReasoningEffort::None` omits the field entirely, because sending it to a non-thinking model is an API error. `speed` picks the OpenAI service tier (`flex`, `default`, `priority`) to trade cost against latency; unset keeps the provider's `auto` routing. `verbosity` (`low`, `medium`, `high`) controls response length and defaults to the provider's `medium`. `parallel_tool_calls` controls whether the model may issue several tool calls in one turn and, when unset, leaves the provider default in place. `metadata` holds up to 16 key-value pairs such as `session_id`, `agent_id`, and `org_id` for usage correlation in the provider dashboard. `extra_headers` adds custom HTTP headers to every request, for gateways and tenant routing; they are applied last and override provider-configured headers.

Every call reports exact prompt, completion, and total token counts in `LlmCompletionMetadata`, including from streams, because the driver requests `stream_options.include_usage`. Provider-reported counts override the local estimate.

### GPT-6 Astra validation

Requests bound for GPT-6 Astra are checked before they leave the process. `validate_config` looks up the OpenAI model profile: a reasoning effort outside the profile's accepted values fails with `AgentLoopError::Configuration("Reasoning effort '{effort}' is unsupported by {model}")`, and Astra accepts Low through Max while rejecting None and Minimal. Setting a temperature on Astra fails the same way with "temperature is unsupported by {model}". Models without an OpenAI profile, `gpt-5.5` for example, accept any value. `validate_body` then inspects the serialized request: `temperature`, `top_p`, and `top_logprobs` are rejected as unsupported (null values are tolerated), `logprobs` is rejected, and on the Chat Completions path any tools or tool messages fail with "GPT-6 Astra tool calling requires the Responses API".

Prompt-cache options are attached only to models that honor them, resolved through `supports_cache_options(model)` to the profile keys `openai/gpt-6-astra`, `openai/gpt-5.6-sol`, `openai/gpt-5.6-terra`, and `openai/gpt-5.6-luna`; aliases and short names resolve to the same key. Residency is checked as well. On the EU data-residency endpoint `https://eu.api.openai.com/v1`, Astra must use Standard processing, so `service_tier` values `fast` and `priority` are rejected with "GPT-6 Astra requires Standard processing with EU data residency" and `default` passes. Residency follows the configured endpoint host, never a model name or locale.

## The OpenAI Responses protocol driver: streaming, reasoning replay, stateful chaining, prompt caching, compaction, tool search, native async tools

### Streaming, reasoning, and state

`OpenResponsesProtocolChatDriver` is the vendor-neutral Responses driver that `OpenAIChatDriver` builds on. It talks to any endpoint that speaks the Open Responses spec (https://www.openresponses.org/), which today includes OpenAI, Azure, OpenRouter, and local models. As with the Chat Completions driver, endpoint and auth are held by the `Provider`.

````rust
use everruns_provider::{LlmRetryConfig, OpenResponsesProtocolChatDriver};

let driver = OpenResponsesProtocolChatDriver::new()
    .with_retry_config(LlmRetryConfig::aggressive());
````

Requests always stream, and answer text arrives as `TextDelta` events. Your system messages are concatenated and delivered as the Responses `instructions` field, never as input items. Images, audio, and files are sent as `input_image`, `input_audio` (wav), and `input_file` parts. Streamed tool-call argument fragments are assembled into complete `ToolCall` values before you see them, and tool results go back as `function_call_output` items, with any images or files inside them dropped with a warning. Finish reasons are neutral strings: `stop`, `tool_calls`, `length`, `error`, `cancelled`, and `unknown` for provider-specific statuses. `LlmCompletionMetadata` reports token counts plus `cache_read_tokens`, `cache_creation_tokens`, and `provider_cost_usd`, where prompt tokens exclude cached reads and cache writes.

Setting `reasoning_effort` on o-series and GPT-5 models turns reasoning on; effort `none` omits the block entirely. The reasoning summary streams as `ReasoningDelta { summary: true }` on its own channel and is never mixed into assistant text. Across turns the reasoning chain persists without server-side state: the driver requests `reasoning.encrypted_content`, stores each reasoning item as an opaque `ReasoningContentPart` keyed by the provider's item id, and replays it before the message it belongs to. Raw chain-of-thought is never persisted, and artifacts from another provider are never replayed to OpenAI after a mid-session provider switch.

The driver defaults to stateless and replays the full transcript, which keeps history intact on OpenRouter and other stateless gateways. `with_stateful_responses(true)`, which `OpenAIChatDriver` sets for you, chains turns through `previous_response_id` and sends only the delta: tool outputs plus new user input. A request never mixes `previous_response_id` with prior transcript input, and when a stateful provider loses its continuation state the driver performs one repaired stateless retry. Rate limits and transient 5xx errors are retried with exponential backoff that honors `x-ratelimit-reset-*` and `retry-after`, auth is re-resolved on every attempt, and an auth failure is never retried. Provider-specific fields, headers, and error classification plug in through `with_request_extension(Arc<dyn OpenResponsesRequestExtension>)`, which is how the OpenRouter crate adds routing without forking the driver.

### Caching, compaction, tool search, and native async tools

The `prompt_cache_key` the driver sends is stable per session: a SHA-256 of strategy, model, cache family, instructions, and tools, prefixed `everruns:` and capped at 64 characters. The cache family is the first of `session_id`, `agent_id`, `harness_id`, or `org_id` found in metadata, so turn-by-turn input never changes the key. Enable it with `config.prompt_cache.enabled`. The explicit strategy adds cache breakpoints with a 30-minute TTL on the models `supports_cache_options` admits, moves instructions into a leading developer message, and disables `previous_response_id` chaining.

The protocol driver exposes compaction as `compact`, which takes a `CompactRequest` and returns output that becomes the next request's input.

````rust
let compacted = driver
    .compact(&endpoint, CompactRequest {
        model,
        input,
        previous_response_id,
        instructions,
        reasoning_state,
    })
    .await?;
// compacted.output becomes the next request's input
````

Query parameters on the endpoint, such as `api-version=preview`, are preserved. When `reasoning_state` is present, the driver instead posts a `compaction_trigger` item to `/responses` with `max_output_tokens` 20000, which requires native Astra Responses. Reasoning effort can also change mid-conversation: a pending effort in `reasoning_state` becomes an in-band `configuration_update` item at the delta boundary, and consecutive updates are collapsed. Both features are gated to `gpt-6-astra`.

Large tool sets scale through OpenAI's hosted `tool_search`. With `with_native_features(true, true)` on the driver (the two flags are native phases and hosted tool search) and `ToolSearchConfig { enabled, threshold }` on the call, tools above the threshold are deferred and grouped into namespaces the model searches server-side; below it, the plain tool list is sent. Tools marked never-deferrable stay always visible. The `tool_search` builtin in chapter 07 sets this up.

Native async tools let the model keep working while a slow lookup runs. `everruns_openai::async_tools::NativeAsyncTools::default().function("web_fetch")` names a registered tool and the driver emits `"async": true` on its wire definition; `.custom(name, format)` turns a tool into an OpenAI `custom` tool; `.continuation(delivery)` delivers results from an earlier turn. Attach the value with `OpenAIChatDriver::with_native_async_tools(options)`. The constraints are strict: only `gpt-6-astra` and `gpt-6-astra-*` models (elsewhere async-marked tools silently stay synchronous), no tool search, and every async tool must be registered, directly callable, automatic policy, and read-only. Pending calls move through queued, running, ready, and delivered inside a serializable `NativeAsyncCheckpoint`. Opt in only when your host consumes native call events with a durable coordinator, which chapters 07 and 08 describe.

## Anthropic, AWS Bedrock, and Google Gemini

### Anthropic

Anthropic is the first of the driver crates. `everruns-anthropic` goes next to `everruns` in `Cargo.toml`, and because it implements the same `ChatDriver` contract, moving an agent from OpenAI to Claude is a configuration change: the agent definition, from its prompts to its capability grants, runs unchanged. A migration guide is at https://docs.everruns.com/how-to/migrate-providers/.

````toml
[dependencies]
everruns = "0.20.0"
everruns-anthropic = "0.18.6"
````

````rust
let api_key = std::env::var("ANTHROPIC_API_KEY")?;
let agent = Agent::builder()
    .instructions(include_str!("../prompts/reviewer.md"))
    .provider(everruns_anthropic::provider("anthropic", api_key))
    .model("claude-sonnet-5")
    .build()?;
````

The label you pass as the first argument becomes the provider id. The key comes from the Anthropic Console, and the example apps read `ANTHROPIC_API_KEY` from the environment themselves. Underneath, the driver id is `anthropic`, `AnthropicChatDriver::new()` takes no arguments, and hosts register the crate with `register_driver(&mut registry)`. Each driver crate also re-exports `ChatDriver` and `DriverRegistry`, so the common case needs no direct `everruns-provider` dependency.

Claude responses stream over server-sent events, Anthropic tool use is mapped onto your typed tools automatically, and extended thinking is adaptive on recent Claude families and budget-based on older ones. Prompt caching needs no setup either: the driver places bounded `cache_control` breakpoints on stable sections of the request. Available models come from Anthropic's `/v1/models` endpoint, which returns capability metadata. One wire requirement is handled for you: `max_tokens` is mandatory on every Anthropic request, so the framework resolves it from the model profile, falls back to a safe default, and retries once with a lower limit if a stale profile causes a rejection.

### AWS Bedrock

Bedrock's credential is a set of AWS fields rather than a single API key, so its constructor takes a typed value instead of a string.

````rust
use everruns_bedrock::{BedrockCredential, provider};

// The driver consults no ambient AWS environment or profile chain; the
// application reads its own variables and passes every field explicitly.
let credential = BedrockCredential::new(
    std::env::var("AWS_ACCESS_KEY_ID")?,
    std::env::var("AWS_SECRET_ACCESS_KEY")?,
    "us-east-1",
);
let bedrock = provider("bedrock", credential);
````

`BedrockCredential::new(access_key_id, secret_access_key, region)` holds an access key id, a secret access key, and a region, and `.with_session_token(..)` adds the optional session token for temporary credentials; no ambient AWS environment or profile chain is consulted. The region defaults to `us-east-1`, an empty region falls back to it too, and any base URL is ignored because the AWS SDK resolves endpoints from the region. A missing field produces "Bedrock provider is missing the AWS access key ID. Configure access_key_id and secret_access_key in provider settings.", and Debug output redacts every secret. The crate speaks the Bedrock Runtime `ConverseStream` API under the driver id `bedrock`, display name "AWS Bedrock", and registers into a driver registry with one call.

Any Converse-compatible model is selected by passing its id as the call's model; the driver neither restricts nor discovers models. Typed tools become Bedrock tool specifications with JSON input schemas, and streamed tool-use chunks come back as parsed tool calls. Conversations are mapped onto system blocks and strictly alternating user and assistant messages, with consecutive tool results grouped and same-role neighbors merged. Images must be base64 data URLs (JPEG, PNG, GIF, or WebP), files must be `application/pdf`, audio is dropped with a warning, and context-window overflow surfaces as a distinct request-too-large error so the loop can compact instead of failing generically.

### Google Gemini

Gemini needs no OAuth setup: an API key from Google AI Studio travels as the `x-goog-api-key` header, and the `everruns-gemini` crate adds implicit and explicit context caching on top of the neutral contract.

````rust
use everruns_gemini::provider;

// GEMINI_API_KEY follows the <UPPERCASE_ID>_API_KEY convention for the `gemini` driver id.
let api_key = std::env::var("GEMINI_API_KEY")?;
let gemini = provider("gemini", api_key);
````

The base URL defaults to `https://generativelanguage.googleapis.com/v1beta`. Point it elsewhere with `.base_url(..)` for a custom or proxy endpoint and model discovery is skipped; against the default host, discovery returns generative `gemini*` models tagged `chat`. The driver id is `gemini`, and hosts register it with one call at startup.

Responses stream through the neutral event channel and tool definitions become `functionDeclarations`. Setting `reasoning_effort` on the call enables thinking and streams thoughts back as reasoning events; effort `none` omits thinking entirely. On thinking models, thought signatures are captured and replayed on the function call they belong to, so multi-turn tool use stays coherent. Images may be base64 data URLs or HTTP URLs; audio degrades to the placeholder `[Audio content not supported]`. Multiple system messages fold into one `system_instruction`. When `max_tokens` is omitted, the output cap comes from the model profile (65536 for `gemini-3.1-pro-preview`) and falls back to 8192.

Supplying a cached-content resource name such as `cachedContents/active` in the call's `prompt_cache` config reuses an explicit Gemini cache; without one, Gemini's implicit caching applies on supported models. The driver also quietly strips the unsupported `additionalProperties` keyword from tool schemas recursively and wraps non-object tool results as `{"result": value}`. Tool calls are released only after a `STOP` finish, so partial or safety-blocked generations never trigger tool execution.

## Fireworks, Microsoft MAI, and Meta

### Fireworks

One Fireworks key exposes the whole serverless catalog of open models (Llama, Qwen, DeepSeek, GLM, Kimi, gpt-oss), served over Fireworks' OpenAI-compatible Chat Completions API. `FireworksChatDriver` is the shared `OpenAIProtocolChatDriver` with Fireworks-specific model discovery added; nothing else in the chat path is vendor-specific.

````rust
use everruns_fireworks::provider;

let api_key = std::env::var("FIREWORKS_API_KEY")?;
let fireworks = provider("fireworks", api_key);
````

The default base URL is `https://api.fireworks.ai/inference/v1`, with the key sent as a bearer token; the exported constant `FIREWORKS_DEFAULT_API_URL` holds the full Chat Completions URL `https://api.fireworks.ai/inference/v1/chat/completions`. A proxy or self-hosted gateway goes in through `.base_url(..)` or at registration time. Keys come from the Fireworks API keys page, and calling without one produces a typed authentication error before any network call. The driver id is `fireworks`, selectable alongside other providers once registered.

Model ids are namespaced, for example `accounts/fireworks/models/llama-v3p1-70b-instruct`, and the last path segment is used as the display name. Streaming, tool calling, vision, and structured output work through the uniform driver like every other provider. At sync time, `/models` metadata is parsed into per-model capability profiles for tool calling, image input, and context window, where the context length doubles as the output cap; only chat models are imported, so image-generation endpoints are filtered out, and only `api.fireworks.ai` is ever contacted, never a proxy or look-alike domain.

### Microsoft MAI

Microsoft MAI models such as `mai-code-1-flash` are served from your own Azure AI Foundry resource, so the constructor takes the resource URL as a second argument.

````rust
use everruns_mai::{MaiAuth, provider};

// MAI_API_KEY follows the <UPPERCASE_ID>_API_KEY convention for the `mai` driver id.
let foundry_key = std::env::var("MAI_API_KEY")?;
let mai = provider(
    "mai-prod",
    "https://my-resource.services.ai.azure.com/openai/v1",
    MaiAuth::ApiKey(foundry_key),
);
````

The URL has the form `https://<resource>.services.ai.azure.com`. A bare resource endpoint is normalized to `/openai/v1` automatically, with query parameters such as `api-version` preserved, so `endpoint().url("chat/completions")` resolves to `.../openai/v1/chat/completions`. `MaiChatDriver` wraps the shared Chat Completions protocol driver under the driver id `mai`, display name "Microsoft MAI", chat only, and registers with one call.

`MaiAuth::ApiKey(key)` sends a plain Azure AI Foundry resource key as an `api-key` header. The alternative, `MaiAuth::EntraOAuth(EntraOAuthConfig { tenant_id, client_id, client_secret, scope, authority })`, authenticates a Microsoft Entra ID service principal through client-credentials OAuth and sends the minted token as a bearer. `scope` and `authority` default to `https://cognitiveservices.azure.com/.default` and `https://login.microsoftonline.com` and may be overridden for sovereign clouds; the allowed authority hosts are `login.microsoftonline.com`, `login.microsoftonline.us`, and `login.chinacloudapi.cn`. Tokens are cached and refreshed 120 seconds before expiry, concurrent refreshes mint one token, and OAuth serves both chat and model sync. When embedding in-process, Entra credentials can also arrive through provider metadata extras, optionally tagged `"auth": "entra"`; an Entra block in extras takes precedence, then typed credential fields, then `api_key`.

With no credentials at all the driver refuses rather than guesses: the error reads "Microsoft MAI provider is not authenticated: configure an Azure AI Foundry API key, or Entra ID OAuth credentials (tenant_id, client_id, client_secret)." A partial Entra config is rejected even when a valid API key is present, and secrets never appear in the message. Discovery requests `<base>/openai/v1/models` with the same credentials as chat, filters out embeddings, speech, TTS, image, and rerank deployments, and treats a project-scoped endpoint with no catalog as "not supported" rather than a sync failure, while 401 and 500 still surface. Non-Azure hosts are never probed, so a custom proxy never receives credentials.

### Meta

Unlike Fireworks and MAI, Meta's Model API speaks the OpenAI-compatible Responses protocol, so `MetaChatDriver` builds on the shared Responses driver rather than Chat Completions.

````rust
let api_key = std::env::var("MODEL_API_KEY").or_else(|_| std::env::var("META_API_KEY"))?;
let agent = Agent::builder()
    .instructions(include_str!("../prompts/commander.md"))
    .provider(everruns_meta::provider("meta", api_key))
    .model("muse-spark-1.3")
    .build()?;
````

This is the incident-commander example's form: it reads `MODEL_API_KEY`, falls back to `META_API_KEY`, and aborts startup if both are missing. Keys are created in the Meta Model API dashboard at https://dev.meta.ai/, and the constructor preconfigures bearer auth against the hosted endpoint `https://api.meta.ai/v1`; the constant `META_DEFAULT_API_URL` is `https://api.meta.ai/v1/responses`. A custom base URL points at a compatible proxy and disables discovery, which otherwise imports the Muse models available to your key from `/v1/models`. The driver id is `meta` and the display name is "Meta Model API".

Because `MetaChatDriver` turns on the shared Responses driver's native feature flags and stateful mode, Meta conversations continue through `previous_response_id` with only the new turn sent, and parallel tool calls, reasoning replay, message phases, and hosted tool search are available where Muse model profiles allow them. Muse Spark 1.3 targets long-horizon coding and multi-step agentic work; both 1.3 ids have a 1,048,576-token context window and accept text, images, audio, video, and PDFs. Data sent to `muse-spark-1.3` is not used for training. `muse-spark-1.3-contributor` is used to train Meta models, is rate-limited, and costs a fraction as much; choose it only if your organization accepts the data-use terms. The tier is part of the model id.

## OpenRouter: one key for many models, routing controls, server tools

### One key for many models

One OpenRouter key reaches a large multi-vendor catalog, and because `everruns-openrouter` wraps the shared Responses driver with OpenRouter's OpenAI-compatible Responses API, every call can also carry OpenRouter's provider routing controls.

````rust
let api_key = std::env::var("OPENROUTER_API_KEY")?;
let agent = Agent::builder()
    .instructions(include_str!("../prompts/researcher.md"))
    .provider(everruns_openrouter::provider("openrouter", api_key))
    .model("z-ai/glm-5.2")
    .build()?;
````

The research example uses this form, reading its key from `OPENROUTER_API_KEY`. Keys come from https://openrouter.ai/keys, are stored encrypted at rest, and are never returned by the API; hosts can instead offer a one-click "Connect with OpenRouter" OAuth PKCE flow, which authorizes at `https://openrouter.ai/auth` and exchanges the token at `https://openrouter.ai/api/v1/auth/keys`. The default base URL is `https://openrouter.ai/api/v1` with bearer auth, and a proxy set through `.base_url(..)` keeps query strings such as `?route=custom`. The model id is a plain OpenRouter slug in `vendor/model` form, so swapping models is a one-string change; provider access and funded credits are still required. The driver id is `openrouter`, display name "OpenRouter", chat only.

Discovery never leaves `openrouter.ai`, so a custom proxy receives no credentials during it. It returns text-output chat models with vendor ownership inferred from the id prefix (`vendor/chat` is owned by `vendor`), and every model gains a derived capability profile from OpenRouter's `supported_parameters` metadata: reasoning, tool calling, structured output, temperature, attachments, and modalities, with limits taken from the routed top provider. Because of this, any model advertising reasoning gets an effort selector of Low, Medium, High, and Extra High defaulting to Medium, even when the framework ships no built-in profile for it. Per-million-token pricing for input, output, and cache read is shown for each discovered model.

The driver sets `reasoning.exclude` on every request, keeping model reasoning out of responses by default, and forwards `reasoning_effort` per call, including `none` to disable reasoning. Setting `openrouter.http_referer` and `openrouter.x_title` in call metadata attributes your application in OpenRouter's dashboard; they become `HTTP-Referer` and `X-Title` headers and are stripped from the wire metadata. The Everruns session id is forwarded too, so every generation from one session groups together in OpenRouter's dashboard. An HTTP 402 surfaces as a typed, non-retryable billing-pressure error with the reason (`in_flight_budget_exhausted` or `insufficient_credits`) and a retry-after hint. OpenRouter's real per-generation spend from `usage.cost` is recorded as the authoritative actual cost alongside the framework's estimate, and budgets debit the actual cost when present.

### Routing controls and server tools

Routing options attach to any call as a typed `OpenRouterRoutingConfig` stored in `LlmCallConfig::driver_options` under the key `openrouter/routing`. Direct OpenAI and every other provider ignore the key. The routing types live in the crate's public `options` module.

````rust
use everruns_openrouter::options::{OpenRouterRoutingConfig, insert_routing_option};

let routing = OpenRouterRoutingConfig::fallback_models(["z-ai/glm-5.2", "backup/model"]);
insert_routing_option(&mut config.driver_options, &routing);
````

`fallback_models` serializes to `{"models": [...], "route": "fallback"}` and the first entry must equal the call's primary model. `OpenRouterProviderRouting` fine-tunes candidate selection with ordered preference, `only` and `ignore` lists, `allow_fallbacks`, `require_parameters`, a `data_collection` policy of allow or deny, `zdr`, `enforce_distillable_text`, `quantizations`, a `sort` by price, throughput, or latency, and a `max_price` in USD per million tokens. Presets express intent instead: `CheapestWithTools` compiles to `{"require_parameters": true, "sort": "price"}`, `NoDataCollection` to `{"data_collection": "deny"}`, and `ZdrOnly`, `ByokFirst`, `LowestLatencyReview`, `StrictJson`, `ReasoningRequired`, and `MaxPrice` cover the rest. Presets combine in list order, explicit provider values always take precedence over preset-derived ones, and negative max-price values are rejected. `capacity_strategy` selects `SharedCapacity` (the default), `ByokFirst`, or `ByokOnly`; the last requires at least one upstream provider slug in the `only` list.

Server-side tools run on OpenRouter without the agent loop dispatching them. `OpenRouterServerToolKind` names eight: `web_search`, `web_fetch`, `datetime`, `image_generation`, `apply_patch`, `fusion`, `advisor`, and `subagent`. `OpenRouterServerTool::with_parameters(kind, json)` forwards tool-specific parameters such as `max_results` verbatim, and the only client-visible artifact is server tool usage. The web-search plugin (`OpenRouterWebSearchPlugin { max_results, search_prompt }`) injects results before the model sees the prompt, and the file-reader plugin reads attached files server-side. Enable server tools per agent by granting the OpenRouter Server Tools capability from chapter 07. It is a harmless no-op on non-OpenRouter providers and is rated High risk, so grant it only to agents you trust with outbound web access.

## The neutral `everruns-provider` contract, part 1: `Provider` values, endpoints, authentication, headers, calling a provider, the provider registry

### A provider is a driver plus an endpoint plus auth

Every constructor in the previous sections returned a `Provider`, and every one of them was built the same way. `everruns-provider` is the neutral contract crate that driver authors depend on instead of all of `everruns-core`; it holds the typed ids, the credential form schema, the error taxonomy, the Open Responses wire types, and URL validation. Its central distinction is between a `ChatDriver`, which implements a wire protocol, and a `Provider`, which is a configured service that speaks it. One driver can serve any number of vendor-compatible services without vendor branches in the runtime.

````rust
use everruns::{Agent, BearerAuth, Provider};
use everruns_provider::OpenAIProtocolChatDriver;

// Neither the driver nor the Provider reads the environment; the application
// supplies the key it holds for this gateway.
let api_key = std::env::var("COMPANY_GATEWAY_API_KEY")?;

let gateway = Provider::new("company-gateway", OpenAIProtocolChatDriver::new())
    .base_url("https://gateway.example/v1")
    .header("x-tenant", "tenant-a")
    .auth(BearerAuth::new(api_key));

let agent = Agent::builder()
    .instructions("Use the configured provider.")
    .provider(gateway)
    .model("assistant-v2")
    .build()?;
````

`Provider::new(id, driver)` takes an arbitrary string id and an owned driver, and `Provider::from_driver(id, arc)` takes a shared `Arc<dyn ChatDriver>`. A new provider has no base URL, no headers, and no auth. `.base_url(url)` sets the endpoint and trims trailing slashes; `.header(name, value)` adds a static header to every request, lowercasing the name, and repeated calls append. `.auth(policy)` attaches an authentication policy, and `.auth_arc(arc)` reuses a pre-built shared one across providers. Because the driver is shared by handle, two providers built with `from_driver` over the same `Arc` can point at different services with different base URLs, headers, and keys.

Behind `.auth(..)` sits the `ProviderAuth` trait, whose `headers` method receives a `ProviderAuthRequest` holding the method, URL, existing headers, and exact serialized body, which is enough for request-signing schemes such as AWS SigV4 as well as plain header auth. `BearerAuth::new(key)` emits `authorization: Bearer <key>`, and `StaticHeaderAuth::new(name, value)` emits one custom header such as `x-api-key` with the name lowercased. Auth runs fresh for every outbound request, so rotating credentials work without extra code, and auth-produced headers override same-named static headers. The `everruns` crate re-exports `BearerAuth`, `Provider`, `ProviderAuth`, `ProviderAuthRequest`, `ProviderEndpoint`, `ProviderKey`, and `StaticHeaderAuth`.

The `ProviderEndpoint` the driver receives composes operation URLs with `url(path)`. It never duplicates an already-present trailing segment (base `.../v1/chat` plus `chat` stays `.../v1/chat`) and merges query parameters from the base URL and the operation, such as a gateway token alongside Gemini's `alt=sse`. `resolve(method, url, body)` produces the final authenticated request. Debug output is safe to log: secrets show as `<configured>`, headers show names only, and URLs are stripped of userinfo, query, and fragment. The endpoint type is deliberately not serializable so credentials cannot leak into snapshots.

### Calling a provider and the provider registry

Messages are built once, in provider-agnostic form, and sent to any driver. `LlmMessage::text(role, text)` builds a plain message and `LlmMessage::parts(role, parts)` a multimodal one, with roles `System`, `User`, `Assistant`, and `Tool`. `LlmContentPart::text`, `image(url)`, `audio(url)`, and `file(url, filename)` accept data URLs or HTTP URLs. `fold_system_messages` joins several system messages with blank lines for drivers that use a dedicated system field, and `prepend_text_prefix` injects an actor label such as `[Alice] ` into the first text part without touching media parts.

````rust
let messages = vec![LlmMessage::text(LlmMessageRole::User, "Hello")];
let config = LlmCallConfig::new("assistant-v2");

let mut stream = gateway.chat_completion_stream(messages.clone(), &config).await?;
let response = gateway.chat_completion(messages, &config).await?;
let catalog = gateway.list_models().await?;
````

`chat_completion_stream` and `chat_completion` delegate to the driver with the provider's own endpoint, so you never pass it by hand; `chat_completion_non_streaming` uses the driver's native single-response path after you check `supports_native_non_streaming()`. `list_models()` returns `None` when the driver does not support listing. Every error from a provider, at request start, on a stream item, from listing, or from compaction, is prefixed `provider '<id>': ...`. `into_boxed_driver()` converts the provider into a self-contained driver with endpoint and auth built in, and `bind_embeddings(driver)` binds an embeddings driver to the same endpoint and credentials. Driver-specific options pass through `LlmCallConfig::driver_options` under namespaced keys that the neutral crate never interprets. Per-request `extra_headers` override protocol, provider, and auth headers by name, case-insensitively and without duplicates; connection-level headers (`host`, `content-length`, `transfer-encoding`, `connection`, `upgrade`) are ignored with a warning.

A `ProviderRegistry` collects configured providers keyed by `ProviderKey`, an open string normalized by trimming and lowercasing, so ` Gateway-PROD ` and `gateway-prod` are the same key. `register(provider)` fails with "provider '<id>' is already registered; use replace() to overwrite intentionally" on a duplicate, including a case variant. `replace(provider)` swaps a configuration at runtime and returns the previous instance while existing handles keep the old configuration. `get(&ProviderKey::new(id))` tolerates casing and whitespace, and `ids()` lists registered ids in sorted order.

## The neutral `everruns-provider` contract, part 2: `DriverId`, driver registry, credentials, model specs, model profiles, discovery, typed ids, `ProviderStore`

### Driver ids, the driver registry, and credentials

A `DriverId` names which driver implementation a provider uses. It is an open string, not a closed enum: any normalized string is valid. Eleven built-in drivers have associated constants whose wire ids are `openai`, `openrouter`, `azure_openai`, `openai_completions`, `anthropic`, `gemini`, `llmsim`, `bedrock`, `mai`, `fireworks`, and `meta`. `DriverId::external("OpenAI-Codex")` creates an embedder-defined id, trimmed and lowercased to `openai-codex` so registration and lookup match regardless of casing.

Hosts that create drivers from stored configuration assemble a `DriverRegistry` at startup. The neutral crate knows nothing about specific implementations; each provider crate registers itself.

````rust
use everruns_provider::{DriverId, DriverRegistry, ProviderConfig};

let mut registry = DriverRegistry::new();
everruns_anthropic::register_driver(&mut registry);
everruns_openai::register_driver(&mut registry);

// The registry never reads the environment. A dev entry point reads the
// variable itself; a production host passes the key it decrypted from its store.
let api_key = std::env::var("OPENAI_API_KEY")?;
let config = ProviderConfig::new(DriverId::OpenAI)
    .with_api_key(api_key)
    .with_base_url("https://gateway.example/v1");
let driver = registry.create_chat_driver(&config)?;
````

`ProviderConfig` is non-exhaustive, so build it with `ProviderConfig::new` and the chained `with_api_key`, `with_base_url`, `with_metadata`, and `with_request_options` setters. `create_chat_driver` never reads environment variables; the host decrypts keys and passes them in. A provider with no credentials still yields a driver, but it rejects every operation locally with "API key is required. Configure the API key in provider settings." before any network I/O.

`registry.register(DriverId::LlmSim, factory)` registers a chat-only factory for a built-in id, `register_external("CUSTOM", factory)` registers an external id with an empty credential schema, and `register_descriptor(DriverDescriptor { id, display_name, services, credential_schema, oauth, chat, embeddings })` declares everything in one unit. Registering the same driver twice panics with "driver already registered for provider ..."; the replace variants swap intentionally, which is how tests substitute the simulator for a real driver. A runtime provider registered directly with `register_provider` takes precedence over descriptors. Embedding providers implement `EmbeddingsDriver` and register through a descriptor's `embeddings` factory.

Credentials reach drivers through the `CredentialProvider` trait, a single `resolve(&DriverId) -> Option<ProviderCredentials>` method that any vault or config source can implement. `EnvCredentialProvider::new()` is the sanctioned pattern for CLIs, dev entry points, and standalone embedders. It reads `<UPPERCASE_ID>_API_KEY` and `<UPPERCASE_ID>_BASE_URL` derived from the driver id, so `anthropic` reads `ANTHROPIC_API_KEY` and `ANTHROPIC_BASE_URL`, and a driver named `custom-driver.v2` reads `CUSTOM_DRIVER_V2_API_KEY` and `CUSTOM_DRIVER_V2_BASE_URL`. The `openai_completions` driver falls back to `OPENAI_API_KEY` and `OPENAI_BASE_URL` when `OPENAI_COMPLETIONS_API_KEY` or `OPENAI_COMPLETIONS_BASE_URL` is unset. An empty value counts as absent, and a proxy can set only the URL. The fields a driver needs are declared as a typed `CredentialFormSchema`; `CredentialFormSchema::api_key(instructions_markdown)` produces one required password field named `api_key` labeled "API Key", while Bedrock and MAI declare discrete typed fields and the simulator declares an empty schema.

### Model specs, profiles, discovery, typed ids, and the provider store

A `ModelSpec` selects a model by naming the runtime provider that serves it and the model name, with no credentials or endpoints attached. This is the execution-facing value the agent builder constructs for you from `.provider(..)` and `.model(..)`.

````rust
use everruns_provider::ModelSpec;

let model = ModelSpec::on("company-gateway", "assistant-v2");
assert_eq!(model.provider.as_str(), "company-gateway");
assert_eq!(model.model, "assistant-v2");
````

A spec serializes as exactly `{"provider": ..., "model": ...}`; stray credential or endpoint fields are silently dropped. It resolves against the provider registry, and an unknown key is a structured `UnknownProvider` error: "provider 'missing' is not registered; registered providers: [east, west]".

`get_model_profile(&DriverId, model_id)` returns what the framework knows about a model, and `estimate_cost_usd(&DriverId, model_id, input, output, cache_read, cache_creation)` prices a generation from the built-in table without calling the provider. Profile types come from the dependency-light `everruns-model-profiles` leaf crate and are re-exported. A stable profile key has the form `{vendor}/{canonical_id}`, for example `openai/gpt-6-astra`, and aliases resolve to one profile.

Discovery distinguishes a driver with no catalog to offer (fall back to curated suggestions) from a catalog and from a failed request. `discover_provider_models(&registry, &config)` fetches a configured provider's live catalog, and `search_provider_models(&registry, &providers, "the luna model")` resolves a fuzzy name to exact provider-qualified ids across every configured provider. Discovered models merge with the static profiles, so bare ids gain names (`gpt-5.5` becomes "GPT-5.5"), newer models work before a profile ships, and hardcoded profiles take precedence for cost data. Connection-level `ProviderRequestOptions { headers, cache_diagnostics }` place gateway headers such as `x-gateway-tenant` next to the base URL and credentials instead of on every agent, and `ProviderTraceConfig` configures per-provider dashboard links. Every entity has a compile-time distinct `TypedId<T>` that displays as `{prefix}_{32 lowercase hex}`, for example `agent_00000000000000000000000000000001`; the prefixes relevant here are `provider` and `model`.

Your own lookup plugs into the host through the `ProviderStore` trait with three async methods: `get_model_spec(model_id)` resolves a typed model id to a credential-free spec (`None` when unknown), `get_default_model_spec()` supplies a store-defined default, and `get_provider_config(&provider_key)` resolves endpoint and auth config immediately before a driver is chosen. Returning `None` from the last is valid only when the provider is registered directly or is intentionally not yet configured, the configure-later workflow that chapter 08 describes.

## Request controls: reasoning artifacts and reasoning updates, execution phases, prompt caching, streaming reconnect, tool schema compatibility

### Reasoning artifacts, reasoning updates, and execution phases

A model's reasoning reaches your application as a `ReasoningContentPart` in the assistant message's content, ordered alongside text and tool calls in the position the provider issued it. The assistant message record itself belongs to chapter 05. The part's `text` is a `ReasoningText` of three kinds: `Plain` is verbatim chain-of-thought from providers that expose it (Anthropic extended thinking, Gemini thought parts, Chat Completions `reasoning_content`); `Summary` is a provider-curated gloss (OpenAI Responses `summary_text`) that is safe to display; `Redacted` means the provider withheld the content but the artifact must still be replayed. `display_text()` returns text for the first two only.

````rust
let part = ReasoningContentPart::opaque("openai")
    .with_item_id(item_id)
    .with_encrypted(encrypted_content)
    .with_tokens(512);

assert!(part.has_replay_state());
let public = part.to_public(); // signature and encrypted stripped, text and tokens kept
````

Each part names the `provider` that produced it and is replayed only against that provider. It may include the provider's `item_id`, an opaque `signature` (Anthropic, Gemini), opaque `encrypted` content (OpenAI), a token count, and `bound_tool_call_id` when a provider scopes a signature to one function call. `to_public()` strips signature and encrypted content for an API surface while keeping the rest. On Anthropic, `ReasoningEffort` maps to a thinking budget through `thinking_budget::from_effort`: Minimal and Low give 1024, Medium 4096, High 16384, Xhigh and Max 32768, and None omits thinking.

`ReasoningState { epoch, baseline, effective, pending }` tracks conversation-scoped reasoning configuration apart from the per-request baseline, so you can see how effort has drifted; the epoch rolls over whenever the model or provider is switched. Native configuration updates are gated by `supports_configuration_updates(model)`, true only for `gpt-6-astra`, and `supported_update_effort` admits Low, Medium, High, Xhigh, and Max as update targets, never None.

`ExecutionPhase` tells an assistant message that is still working (`Commentary`) apart from the final completed answer (`FinalAnswer`). A `PhaseSource` accompanies it: `Provider` means the model reported the phase on the wire, which OpenAI GPT-5.x does, and `Derived` means the runtime inferred it from tool-call presence, a weak signal. Anthropic and Gemini phases are always derived and never sent to the provider. The streamed phase hint advances to a phase exactly once and never reverts; the completed message's phase is authoritative.

### Prompt caching, streaming reconnect, and tool schema compatibility

Per call, `PromptCacheConfig { enabled, strategy, gemini_cached_content }` controls prompt caching: `strategy` is `Auto` (the default) or `Explicit`, and the optional resource is a Gemini `cachedContents/{id}` name. Unsupported providers ignore it without failing; the `prompt_caching` builtin in chapter 07 sets it. For Anthropic specifically, `CacheDiagnosticsConfig { enabled, previous_message_id }` requests the `cache-diagnosis` beta to learn where a prompt prefix diverged, the connection-level `cache_diagnostics` flag on `ProviderRequestOptions` turns it on automatically, and `volatile_suffix_len` (default 0) excludes a volatile conversation tail from the cache breakpoint; the engine sets it to the count of live facts messages it appends.

Streaming reconnect handles a provider that returns 200 and then fails before the first SSE event is decoded: "error decoding response body", truncated bodies, mid-stream resets, and stale pooled keep-alives. Both built-in HTTP drivers re-send the identical request transparently. Custom drivers get the same behavior by supplying a connect closure that performs one full send attempt.

````rust
let (stream, retry_metadata) = connect_sse_with_reconnect(
    &self.retry_config,
    "MyProtocolDriver",
    |attempts| self.send_request(endpoint, &body, attempts),
)
.await?;
````

`connect_bytes_with_reconnect` is the raw byte-stream analogue for drivers that parse SSE by hand, as Gemini does. The call resolves at first-token time and the peeked first item is replayed so nothing is lost. `is_reconnectable_reqwest_error` classifies body, decode, connect, request, and timeout errors as reconnectable, while malformed SSE payloads surface instead of being masked. Header-phase retries and body-phase reconnects share one budget bounded by `LlmRetryConfig.max_retries`, with backoff tuned through `initial_backoff`, `max_backoff`, `backoff_multiplier`, and `jitter_factor`, and total time capped by `max_retry_elapsed`. A provider that returns 200 and then sends nothing fails after a fixed 120-second first-item stall, which is neither retried nor configurable.

Tool schema compatibility runs at the request boundary without editing your tool definitions. `sanitize_openai_tool_schema(schema)` produces an OpenAI-acceptable copy: the standard Zod email pattern emitted by Zod-based MCP servers such as Resend is rewritten to a lookaround-free equivalent that still rejects the same invalid addresses, and any other lookahead or lookbehind pattern is dropped from the model-facing copy only, with "Validation for this value is enforced by the tool." appended to the property description. `strict_openai_tool_schema(schema)` derives a schema for OpenAI strict structured outputs: every property is listed as required, originally optional ones become nullable, and every object level gets `additionalProperties: false`. It returns `None` rather than rewriting lossily whenever the schema uses anything outside `type`, `properties`, `required`, `additionalProperties`, `items`, `enum`, `anyOf`, `description`, and `title`; callers then fall back to the sanitized non-strict schema.

## Provider errors, user-facing error codes, and URL safety (SSRF protection)

### Semantic error kinds and the agent loop error

When an LLM call fails you match on a semantic kind instead of re-parsing strings. `LlmErrorKind` is assigned by the driver where the HTTP status and body are still available, and the framework's retry logic treats it as authoritative.

````rust
use everruns_provider::{AgentLoopError, LlmErrorKind};

match (&error, error.llm_error_kind()) {
    (_, Some(LlmErrorKind::RateLimited | LlmErrorKind::Unavailable)) => retry_later(),
    (_, Some(LlmErrorKind::QuotaExhausted)) => page_operator(),
    (_, Some(LlmErrorKind::Authentication)) => rotate_credentials(),
    (AgentLoopError::RequestTooLarge(_), _) => compact_history(),
    _ => report(&error),
}
````

The kinds are `Authentication` (401/403, bad API key), `QuotaExhausted` (out of credits, which needs operator action), `BillingPressure { reason, retry_after_secs }` (reason `in_flight_budget_exhausted` or `insufficient_credits`), `RateLimited` (a transient 429), `Unavailable` (5xx, 529, or network failure), `AttestationRequired` (an account gate such as OpenRouter's 18+ confirmation, kept apart from a credential failure even though both arrive as 403), `InvalidRequest` (any other 4xx), and `Other`, which falls back to string classification. `is_transient_llm_error()` returns true for rate limited and unavailable only, and the semantic kind overrides conflicting message text. Quota exhaustion is detected from the body before the status is consulted, because providers surface it under different statuses (OpenAI 429 `insufficient_quota`, Anthropic 400 "credit balance is too low"), and it is never retried. `LlmError` also records how many retries a lower layer already consumed so the reason loop does not multiply attempt budgets.

`AgentLoopError` is the single error type across the loop. Beyond `Llm`, its variants include `RequestTooLarge(String)`, `ModelNotAvailable(String)`, `ModelNotConfigured`, `ToolExecution`, `MessageStore`, `EventEmission`, `Configuration`, `MaxIterationsReached(usize)`, `Cancelled`, `NoMessages`, `AgentNotFound`, `HarnessNotFound`, `SessionNotFound`, `Internal`, and `DriverNotRegistered(String)`. Context-window overflow is a dedicated `RequestTooLarge` that is never retried, so callers can compact history. `ModelNotAvailable` includes the requested model id, and tier-gated 403s classify as model unavailable rather than misconfigured auth. `is_non_retryable()` identifies the deterministic failures: not-found lookups, no messages, model not configured, configuration, and driver not registered.

Custom drivers construct errors with `AgentLoopError::llm_kind(kind, msg)` for an explicit kind or `AgentLoopError::llm(msg)` for an untyped one classified downstream. `LlmErrorKind::from_provider_code("insufficient_quota")` maps a provider's machine-readable code, `from_provider_status(status, body)` maps an HTTP status plus body, and `from_error_text(text)` handles SDKs such as Bedrock that expose no status. `is_request_too_large(status, text, extra_patterns)` and `is_model_not_found(status, text, patterns)` in the driver helpers cover the two dedicated variants, with pattern constants such as `ANTHROPIC_TOO_LARGE_PATTERNS` supplied.

### User-facing error codes

Any agent loop error yields a safe display message through `user_facing_message()`, and a structured `UserFacingError` with a stable code and safe fields through `user_facing_error(context)`. Raw provider text never reaches end users on this path; unknown input maps to `processing_error`.

````rust
let context = UserFacingErrorContext::default()
    .with_provider("openai")
    .with_model_id("gpt-5.6-terra")
    .with_retry_after(7);
let public = error.user_facing_error(context);
// {"code":"provider_rate_limited","fields":{"model_id":"gpt-5.6-terra","provider":"openai","retry_after":7}}
````

The `codes` module defines the vocabulary: `budget_exhausted`, `budget_paused`, `model_unavailable`, `model_not_configured`, `request_too_large`, `provider_rate_limited`, `provider_usage_limit_reached`, `provider_misconfigured`, `provider_quota_exhausted`, `provider_attestation_required`, `provider_unavailable`, `processing_error`, `dependency_unavailable`, `invalid_tool_schema`, `max_iterations`, `soft_limit_reached`, and `blocked_by_hook`. Every code has an English `fallback_message()`; `request_too_large` renders "The conversation has become too long for the model to process. Please start a new session or reduce the context size." and the default is "I encountered an error while processing your request. Please try again later." An attestation gate points the account holder at the confirmation page, `https://openrouter.ai/settings/preferences` by default, with hostile payloads bounded. `classify_runtime_error_message(text, &context)` turns any raw error string into the same structured form, and `apply_to_message_metadata` writes `error_code` and `error_fields` onto a message while clearing stale fields from an earlier error. Fields serialize in deterministic key order. The budget, usage-limit, and blocked-by-hook codes come from builtins in chapter 07.

### URL safety

Custom provider base URLs and MCP endpoint URLs pass through SSRF validation. `validate_safe_url(raw)` is the static check for write time: it verifies scheme, hostname patterns, and IP ranges with no DNS resolution. `validate_url_dns_pinned(raw)` is the execution-time check, run just before each outbound call: it resolves the hostname to catch DNS rebinding, rejects the whole resolution if any address is blocked, and returns the resolved socket addresses so the connection can be pinned to those IPs. Lookup is capped at 5 seconds, and a lookup that errors, times out, or returns nothing is treated as blocked. `is_blocked_ip(addr)` checks an already-resolved IP for custom resolvers.

Errors are typed as `UrlValidationError::InvalidUrl`, `DisallowedScheme`, `MissingHostname`, and `BlockedHost`, the last displaying "Blocked host: {host} (private/internal address)". Only `http` and `https` are allowed. The blocklist covers `localhost` in all forms, IPv4 loopback 127/8, 0.0.0.0, obfuscated decimal and hex spellings, RFC1918 ranges, link-local 169.254/16 including the cloud metadata endpoint 169.254.169.254, `metadata.google.internal`, carrier-grade NAT 100.64/10, the documentation ranges, IPv6 loopback `::1`, `::`, link-local `fe80::/10`, unique-local `fc00::/7`, and IPv4-mapped IPv6 addresses. There is no opt-in for private hosts: `EVERRUNS_SSRF_ALLOW_CIDRS` does not unblock private IPs, so a provider endpoint on an RFC1918 or loopback address cannot be allowlisted.

## Custom providers: implementing `ChatDriver`, driver-author helpers, and the extension seam

### Implementing `ChatDriver`

Use a custom provider when your application talks to a model service the framework does not configure for you. The extension boundary is the public `ChatDriver` trait plus a `Provider` value; there is no closed provider enum to extend and no provider-specific branching in application code. Only `chat_completion_stream` is required. Every other method has a default.

````rust
use async_trait::async_trait;
use everruns::{
    Agent, AgentLoopError, BuildError, ChatDriver, LlmCallConfig, LlmMessage,
    LlmResponseStream, LlmStreamEvent, Provider, ProviderEndpoint,
};

struct DownstreamProtocol;

#[async_trait]
impl ChatDriver for DownstreamProtocol {
    async fn chat_completion_stream(
        &self,
        _endpoint: &ProviderEndpoint,
        _messages: Vec<LlmMessage>,
        _config: &LlmCallConfig,
    ) -> Result<LlmResponseStream, AgentLoopError> {
        Ok(Box::pin(futures::stream::iter([
            Ok(LlmStreamEvent::TextDelta("downstream works".to_string())),
            Ok(LlmStreamEvent::Done(Box::default())),
        ])))
    }
}

fn agent_for(driver: impl ChatDriver) -> Result<Agent, BuildError> {
    Agent::builder()
        .instructions("Use the configured provider.")
        .provider(Provider::new("acme", driver))
        .model("assistant-v1")
        .build()
}
````

This minimal driver returns one text delta followed by a `Done` event with default metadata, and the agent builder accepts it as it accepts OpenAI. It needs no credentials because it never leaves the process; a real driver gets its endpoint and auth from the `Provider` it is attached to (`.base_url(..)` plus `.auth(BearerAuth::new(key))` or `.auth(StaticHeaderAuth::new(name, value))`, as in part 1 of the contract). The `everruns` crate re-exports `ChatDriver`, `LlmCallConfig`, `LlmCompletionMetadata`, `LlmMessage`, `LlmResponseStream`, and `LlmStreamEvent`, so a custom driver in application code needs no direct `everruns-provider` dependency. A real driver reads `endpoint.url(path)` and `endpoint.resolve(..)` for its URL and auth, converts the messages and config to its wire format, and maps the response back onto the event vocabulary: `TextDelta(String)`, `ReasoningDelta { delta, summary }`, `ReasoningItem(ReasoningContentPart)`, `ToolCalls(Vec<ToolCall>)`, `NativeToolCall(NativeToolCall)`, `MessagePhase(ExecutionPhase)`, `Done(Box<LlmCompletionMetadata>)`, and `Error(LlmStreamError)`. The enum is non-exhaustive, so consumers ignore variants they do not recognize. Structured errors inside an accepted stream use `LlmStreamError::provider(code, status, message)`, whose `kind()` tries the provider code, then the status, then the text. `LlmCompletionMetadata` follows the disjoint token convention: `prompt_tokens` excludes cached reads, with `cache_read_tokens` and `cache_creation_tokens` additive on top, and `disjoint_prompt_tokens(reported_input, cache_read)` normalizes providers that report inclusively.

`list_models` defaults to `Ok(None)`, meaning unsupported, and the remaining optional methods likewise describe capabilities. `supports_compact` and `compact` offer native compaction, which only OpenAI's Responses API implements today; provider-owned compaction output travels back opaquely and is mutually exclusive with server-side continuation. `supports_stateful_responses` defaults to false and should stay false unless the service persists Responses API state. External drivers should also override `effective_context_window(model)` so the host does not have to estimate it, and `supports_parallel_tool_calls(model)`, which OpenAI and Anthropic answer true and Gemini and Bedrock false. `native_async_driver` returns a specialized driver for native async tool calling while retaining the synchronous fallback.

### Driver-author helpers and the extension seam

The neutral crate's default `http` feature supplies the pieces the built-in drivers are made of: the shared protocol drivers, request helpers, and SSE reconnect. A transport-free consumer sets `default-features = false` and keeps discovery normalization, ranking, matching, and search. The `tls-aws-lc-rs` feature, on automatically with `http`, installs a process-wide TLS crypto provider, and repeated installs are safe no-ops. The `openapi` feature derives OpenAPI schemas for the provider types.

Instead of building a `reqwest::Client` per call, drivers share process-wide SSRF-hardened clients. `shared_streaming_http_client()` is tuned for streaming chat: a 10-second connect timeout, a 300-second per-read inactivity timeout that never caps total stream duration, a 15-second pool idle timeout, redirects disabled, and a DNS-pinned resolver. `shared_request_http_client()` serves bounded calls such as embeddings with a 60-second overall timeout and a 90-second pool idle timeout. Both check every resolved IP against the private blocklist at connection time, failing with "host {host} resolves to blocked address ... (private/internal)", and both return a 3xx `Location` as-is rather than following it.

Beyond transport, `parse_data_url("data:image/svg+xml;base64,PD4=")` splits an attachment into media type `image/svg+xml` and payload `PD4=`, returning `None` on failure rather than assuming JPEG. `AUDIO_CONTENT_PLACEHOLDER` is the standard `[Audio content not supported]` text for providers that reject audio. `fetch_models(request, fetch_err_prefix, parse_err_prefix, none_on_statuses, map)` implements `/models` discovery for an OpenAI-compatible provider in a few lines: an authenticated request, a response type, the statuses that mean "unsupported", and a mapping closure, with each provider keeping its own error wording.

You may not need to write a driver at all. `Provider::new("custom", OpenAIProtocolChatDriver::new())` reuses the OpenAI-protocol driver under a custom provider name for any OpenAI-compatible gateway, and GPT-6 Astra rules apply only when the model resolves to an OpenAI profile. When embedding Everruns as a host, additional custom drivers are registered through the platform definition, which extends the supported-provider table without waiting for upstream support; host composition is the subject of chapter 10.

*2026-09-17 03:24 - claude-fable-5.1*
