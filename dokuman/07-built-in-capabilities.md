<!-- source: everruns/everruns @ 6cf2c15e5 (crate everruns 0.20.0). The working tree fast-forwarded to 7d9e07a0b (0.21.1) during extraction; identifiers and examples were audited against 6cf2c15e5. -->
# Built-in Capabilities

A default `everruns` build already contains the components that turn a bare model loop into a working agent, from context compaction and deferred tool loading to guardrails and skills. All of it ships in the `everruns-builtins` crate behind the `builtins` feature, which is on by default, and none of it is active on an agent until you name it: every builtin has a stable snake_case id, and attaching one is a matter of passing that id, or a typed value that resolves to it, to the builder. What follows is the catalog, grouped by what each builtin does for the agent, with a working example wherever the syntax is documented.

## The builtins feature, bundle registration, and the capability catalog

````toml
[dependencies]
everruns = "0.20.0"
````

````rust
use everruns::prelude::*;

let agent = Agent::builder()
    .instructions("Answer questions about the current schedule.")
    .model(Model::simulated("It is 09:15 UTC."))
    .capability("current_time")
    .capability("message_metadata")
    .build()?;
````

No feature flag changed. The default feature set already includes `builtins`, so the `current_time` and `message_metadata` capabilities are registered in the host and the bare string ids attach them. The `builtins` feature is defined as `builtins = ["dep:everruns-builtins", "everruns-host/builtins"]`, and it is kept optional so that a project with `default-features = false` compiles no built-in implementation package at all. With the feature on, the `everruns` facade re-exports the typed values `AgentInstructionsConfig`, `CompactionConfig`, `CompactionStrategy`, `Skills`, `StatelessTodoList`, and `ToolSearch`; the host registers the default tool set on every turn and installs the output-persistence hook as the final post-tool hook.

A capability is a modular unit that contributes tools and system prompt additions and may declare features, and an agent enables only what it needs. Dependencies resolve at runtime, so you never add one by hand. Capabilities apply in the order configured on the agent, and earlier capabilities' prompt additions appear first, so the order on the builder is also the prompt priority. The Framework offers only capabilities that run through its portable, in-process host contract, which means builtins backed by services the Framework runtime or an explicit application integration supplies, such as files, session storage, current time, compaction, tool search, and skills. Subagents, agent handoff, and user hooks are hosted Platform capabilities; their ids and config shapes are documented alongside the portable ones and appear later in this chapter.

The runtime catalog is a deterministic list of 28 ids in this order: `human_intent`, `infinity_context`, `skills`, `agent_instructions`, `channel_context`, `current_time`, `message_metadata`, `stateless_todo_list`, `btw`, `budgeting`, `self_budget`, `compaction`, `error_disclosure`, `openai_tool_search`, `claude_tool_search`, `tool_search`, `auto_tool_search`, `prompt_caching`, `parallel_tool_calls`, `native_async_tools`, `system_commands`, `tool_output_persistence`, `tool_output_distillation`, `loop_detection`, `progress_guard`, `tool_call_repair`, `prompt_canary_guardrail`, `guardrails`. Every builtin has a display name and a category drawn from Core, Optimization, System or Cost Control, Safety, Session, and Automation, and declares which tools and prompt additions it contributes. Attach one in Rust by passing a bare string id, a `CapabilityRef` with JSON config, or one of the typed values to `.capability(...)`. In JSON agent or harness config, list the id as a bare string in `capabilities` or as `{ "ref": "<id>", "config": { ... } }`; several docs pages spell the object form `capability_ref` instead of `ref`.

Advanced hosts that assemble their own registry call into the bundle directly. `register_portable_capabilities` registers every portable capability in stable product order. `register_runtime_capabilities` registers only the runtime-safe subset that works in the default embedded host. `portable_capability_registry()` creates a fresh `CapabilityRegistry::new()` and registers the portable set in one expression. Registration is explicit; linking the crate has no inventory side effect. `runtime_capability_registry()` from the host crate returns the Framework preset (an empty core registry plus the runtime-safe catalog) when `builtins` is on.

- Registering an id or alias that already exists is rejected rather than replacing an application-provided implementation, and bundle registration is all-or-nothing: a late collision leaves the registry unchanged. The host fails to start with a duplicate catalog.
- The only host-provided dependency the bundle relies on is `session_file_system`, declared by `skills`, `attach_skill`, `tool_output_persistence`, and `tool_output_distillation`.
- The embedded host does not include `usage_limit_auto_continue`, `openui`, `a2ui`, or `openrouter_server_tools`; those belong to service-backed product composition.
- The optional `ui-capabilities` feature on `everruns-builtins` (off by default, no extra dependencies) adds the OpenUI and A2UI prompt capabilities and the `everruns_builtins::{openui, a2ui}` catalogs.
- Two legacy context-free tools, `get_current_time` and `write_todos`, are also installed into the host's executor tool registry by `register_default_tools` so blueprint and seed-time tooling stays compatible; `register_monitor_tools` installs only `get_current_time`.

## Typed presets: CompactionConfig and ToolSearch

````rust
use everruns::{Agent, CompactionConfig, ToolSearch};
use everruns::prelude::*;

let agent = Agent::builder()
    .instructions("Find the right tool and keep long sessions focused.")
    .model(Model::simulated("ready"))
    .capability(CompactionConfig::new().budget_percent(0.85))
    .capability(ToolSearch::automatic())
    .build()?;

let mut session = Engine::new().create(agent);
let turn = session.run("Summarize the plan.").await?;
assert!(turn.success);
````

This is the pattern from the crate README and the capability_configuration example. Two typed values stand in for two capability ids, and a full offline turn succeeds on the in-memory engine. `CompactionConfig::new()` converts into a capability reference with the stable id `compaction` and turns on context compaction with defaults; `ToolSearch::automatic()` converts into the id `auto_tool_search` and turns on model-adaptive deferred tool loading. Both implement `IntoCapability`, so they can be passed anywhere a capability is accepted, and both emit only the fields you set.

The builder methods on `CompactionConfig` are typed spellings of the `compaction` config keys described under Context management. `.strategy(CompactionStrategy::...)` chooses among the `Auto`, `Native`, `ObservationMasking`, and `Summarization` variants of a `#[non_exhaustive]` enum, and `.proactive(bool)` toggles compaction before a provider rejects an oversized request. `.budget_percent(f32)` sets the proactive trigger as a fraction of the model's context budget, and a value outside 0.1 through 1.0 fails at build time, not at run time:

````rust
use everruns::{BuildError, CompactionConfig};

for invalid in [0.05, 1.5, f32::NAN] {
    let err = Agent::builder()
        .instructions("x")
        .model(Model::simulated("x"))
        .capability(CompactionConfig::new().budget_percent(invalid))
        .build()
        .unwrap_err();
    assert!(matches!(err, BuildError::InvalidCapability { ref id, .. } if id == "compaction"));
}
````

`ToolSearch` likewise exposes `.threshold(n)` and `.never_defer([...])`, the deferral threshold and pin list that the `tool_search` entry under Tool discovery explains; the pinned list accepts any iterator of string-likes.

````rust
let agent = Agent::builder()
    .instructions("Tell the time when asked.")
    .model(Model::simulated("09:15"))
    .capability("current_time")
    .capability(ToolSearch::automatic().threshold(1).never_defer(["get_current_time"]))
    .build()?;

let mut session = Engine::new().create(agent);
let context = session.inspect().await?;
// context.tools contains both `tool_search` and the pinned `get_current_time`
````

With the threshold at 1 and a single tool registered, inspecting the session shows `tool_search` alongside the pinned `get_current_time`. Configuration validation for tool search accepts only the keys `threshold` (a non-negative integer) and `never_defer` (a list of valid tool names); an unknown field is rejected with a message naming it. The presets cover the common settings; the nested `compaction` options are reachable through `CapabilityRef` with JSON config.

## Built-in harnesses: HarnessDefinition, Base, Generic, and Data Analyst

````rust
use everruns_core::HarnessDefinition;

let harness = HarnessDefinition::new("generic", "You are helpful.");
````

A harness is the base environment beneath every session, holding a system prompt and a default model along with the capabilities bundled for that environment. Every session is assigned one, and it is the lowest layer of the overlay chain, below the agent and then the session. `HarnessDefinition::new(name, system_prompt)`, exported by `everruns-core` (the crate advanced hosts depend on directly), creates a harness with just those two values and leaves every other field at its default. A blank or whitespace-only prompt is normalized to none, so `""`, `"  "`, and `"\n\t"` all yield no base prompt.

A harness can hold more than a prompt.

- A base system prompt that sits below the agent and session prompts.
- A default model, last in the priority order of controls, then session, then agent, then harness, so it applies only when nothing above it names a model.
- A list of enabled capabilities, each with per-harness configuration. The JSON form is `{"ref": "web_fetch", "config": {"timeout_ms": 30000}}`.
- Starter files copied into each new session.
- A network access list merged with the agent and session layers.
- Embedder metadata folded root to leaf across the chain, with the leaf winning, for example `"embedder_metadata": {"deployment": "local"}`.
- A harness-wide `parallel_tool_calls` boolean preference. An explicit `parallel_tool_calls` field on a harness, agent, or session takes precedence over the `parallel_tool_calls` capability.
- Remote MCP servers under `mcpServers` (alias `mcp_servers`) that descendant layers inherit, such as `{"docs": {"type": "http", "url": "https://docs.example.test/mcp"}}`.

The Base harness, type id `base`, is the blank slate among the documented built-ins: no bundled capabilities, the default prompt "You are a helpful assistant.", and no default model. Choose it when you want full control over which tools are available, or when you want to test one capability in isolation with no default tools present. An agent's own capabilities are layered on top, so an agent declaring `"capabilities": ["web_fetch"]` on the Base harness has web fetch and nothing else.

The Generic harness, type id `generic`, is the recommended default for general-purpose assistants and for coding and research work. It shares the Base prompt and has no default model. It configures 16 capabilities: 14 user-facing defaults plus the cross-cutting helpers `human_intent` for tool narration and `btw` for side questions. The user-facing set covers File System (read, write, list, grep, and delete in `/workspace`), Bashkit Shell, Web Fetch, Storage (key/value plus encrypted secrets), Session (metadata and title), Session Schedules (cron-style wakeups that re-enter a session on a timer), AGENTS.md, Agent Skills, Infinity Context, OpenAI Tool Search (the auto tool search docs say Generic uses the model-adaptive variant), Context Compaction at an 85 percent budget, Budgeting, Self-Budget, and Tool Output Persistence. It enables `parallel_tool_calls` with `mode: "prefer"`. Infinity Context and Context Compaction are on together and keep long sessions effectively unbounded.

The Data Analyst harness, type id `data-analyst`, extends Generic with five data capabilities: Session SQL Database (`sql_execute`, `sql_query`, `sql_schema`; session-scoped SQLite that auto-creates on first write), Persistent Memory (`remember`, `recall`, `forget`; cross-session memory with 8 memories auto-injected per turn), OpenUI (charts, tables, dashboards, and KPI cards inline in chat), Todo List (`write_todos`), and Data Knowledge. The last mounts a `/knowledge/` scaffold with `tables/`, `business/`, and `queries/` directories the agent reads before writing SQL. The harness system prompt is a structured six-step pipeline: Recall, Inspect (verify schema with `sql_schema`), Plan, Execute and Validate (self-correct on zero rows, duplicates, or NULL aggregations), Visualize through OpenUI, and Learn by remembering corrections. The scaffold is a read-only mount that you populate with your organization's curated knowledge.

- `tables/`: one `.md` per table with columns, types, and gotchas.
- `business/`: metric definitions, business rules, and domain terms.
- `queries/`: validated `.sql` files as reusable templates.

## Instructions and session context: AGENTS.md, message timestamps, channel context, current time, human intent, /btw, session tools

### Workspace instruction files

````rust
use everruns::AgentInstructionsConfig;
use everruns::prelude::*;

let agent = Agent::builder()
    .instructions("Follow the project conventions you find in the workspace.")
    .model(Model::simulated("Using the repo conventions."))
    .capability(FileSystem)
    .capability(AgentInstructionsConfig::default())
    .build()?;
````

Capability id `agent_instructions`, display name "AGENTS.md", category Core, no tools. Write an `AGENTS.md` file into the session workspace and the capability reads it every turn and injects it as the leading user message. By default only `/workspace/AGENTS.md` is read; `AgentInstructionsConfig` converts into a reference whose `files` list becomes the config. In JSON, `{ "ref": "agent_instructions", "config": { "files": ["AGENTS.md", "CLAUDE.md"] } }` reads two filenames in that order. `files` accepts 1 to 16 unique names, default `["AGENTS.md"]`; unknown keys are rejected, a leading `/workspace/` or `/` is stripped, and paths containing `..`, empty segments, or trailing slashes fail validation.

Files resolve hierarchically from the filesystem root down to the session working directory, so broad rules apply first and deeper files override them; a sibling directory outside the ancestor chain is never loaded. Each file is wrapped in an `<agent-instructions source="...">` block with angle brackets and ampersands escaped so a file cannot forge a system-prompt block, behind a trust header saying that system instructions take precedence and that deeper files win on disagreement. The block is never system prompt. Limits are 32 KiB per file, truncated with a "[<source> was truncated - content exceeds 32 KiB limit]" marker, and 128 KiB per turn. Blank files are skipped without shadowing a parent, missing files are ignored, and with an invalid runtime config the capability falls back to `AGENTS.md`. The public helpers `format_agents_md_content` and `format_instruction_file_content` produce the same wrapped block from your own code. AGENTS.md is the simpler alternative to skills when only project context is needed.

### Conversation context and session tools

````rust
let agent = Agent::builder()
    .instructions("Help the team in this Slack thread.")
    .model(Model::simulated("On it."))
    .capability("message_metadata")
    .capability("channel_context")
    .capability("current_time")
    .capability("human_intent")
    .capability("btw")
    .build()?;
````

Each of these ids attaches as a bare string and adds one kind of context to the turn.

- `message_metadata` (display "Message Metadata", Core, no tools) prefixes each user and agent message with its UTC timestamp in the LLM request, for example `[time 2026-06-11T09:15:42Z] What changed since yesterday?`. Stored messages are unchanged, system and tool-result messages are never annotated, and timestamps are stable across turns so prompt caching is unaffected. Config key `fields` (array, default `["timestamp"]`) chooses the rendered fields; an empty array disables annotations. A leading `[time ...]` echoed by the model is stripped before persistence.
- `channel_context` (display "Channel thread context", no tools) tells an agent driven by a messaging channel such as Slack who else is in the thread (`Thread participants: Alice, Bob`) and, when reported, where the user is looking. The context is injected as conversation context rather than system prompt, is read from the persisted session key-value store (a per-session store that survives an agent restart), and is absent for sessions that are not channel-backed.
- `current_time` (Core, 1 tool) adds `get_current_time` with optional `timezone` (an IANA name, default `UTC`) and `format` (`iso8601` default, `unix`, or `human`); an unknown zone returns the tool error "Unknown IANA timezone". Independently, the capability injects the current UTC time into every turn as a dynamic fact at the conversation tail, so the model knows "now" without invalidating the system-prompt cache.
- `human_intent` (display "Human Intent") adds an optional `human_intent` string argument, max 120 characters, to every tool schema, asking for an action phrase such as "Listing all harnesses". The narration is shown in every tool-call phase and stripped before the tool runs, and a call that omits the argument runs unchanged with no narration.
- `btw` gives the user `/btw <question>` for a side question about the session. The answer is grounded in the full merged context and is not persisted, and the side answer neither calls tools nor asks follow-ups. A provider failure comes back as a classified non-fatal error such as `provider_rate_limited`.

The session tools, id `session`, category Session, let the agent read and write session-scoped state: `get_session_info` (no parameters; returns session ID, title, and agent name), `write_session_title` (required `title`), plus the key-value and secret storage tools that Generic lists as Storage. Set `auto_title: true` in the config and the agent names the conversation with a concise 3 to 7 word title before the first substantive request, updating it later only when the primary theme materially changes. Title writes update metadata only and emit `session.title.updated` with the previous and new title.

## Cost and progress limits: budgeting, self_budget, usage_limit_auto_continue, loop_detection, progress_guard

````rust
use serde_json::json;

let agent = Agent::builder()
    .instructions("Refactor the module. Stop and report if you are stuck.")
    .model(Model::simulated("done"))
    .capability(FileSystem)
    .capability("budgeting")
    .capability("self_budget")
    .capability(CapabilityRef::new("loop_detection").config(json!({ "threshold": 2 })))
    .capability("progress_guard")
    .build()?;
````

`budgeting` (display "Budgeting", feature `budgeting`, 1 tool) makes the agent aware that its session may have enforced spending limits. It adds `check_budget` (display "Check Budget", no parameters), which returns `status`, a `budgets` array with `currency`, `limit`, `balance`, `soft_limit`, `percent_remaining`, and per-budget `status` (`active`, `warning`, or `exhausted`), and a `hint`. Real data arrives when the host supplies a budget checker on the tool context; without one the tool returns the stable fallback `{"status": "no_budgets", "budgets": [], "hint": "No budgets are configured for this session. You can proceed without budget constraints."}`. A fixed prompt section tells the model to check budget before expensive work.

`self_budget` (category System, prompt-only, no tools) instructs the agent to treat an indicative budget stated in chat, such as "you have $7", as a soft agent-managed target. Usage data comes from `get_session_info`. The injected "Self-Managed Budget" section says to track spend around expensive phases, qualify estimates when pricing is partial, tighten scope and output as the target nears, and never report the target as a platform budget. Choose `self_budget` when the budget comes from conversation and `budgeting` when it comes from configuration and must be enforced; they combine.

`usage_limit_auto_continue` (Core, Risk Low, no tools) resumes a session on its own after a provider plan's usage limit resets instead of leaving it idle for hours. The bare id gives the defaults: continuation fires 120 seconds after the reported reset with the prompt "Continue tasks". Config keys are `delay_seconds` (integer 0 to 86400, default 120) and `prompt` (string, default "Continue tasks"); for example `{ "ref": "usage_limit_auto_continue", "config": { "delay_seconds": 300, "prompt": "Resume the migration where you left off." } }`. It reacts only to the `provider_usage_limit_reached` error class with a concrete `resets_at` timestamp; ordinary rate limits are ignored. The continuation is a one-shot session schedule, a timed wakeup stored by a schedule store and fired by a poller, at `resets_at + delay_seconds`, and it injects the prompt as the next user message. Without a schedule store and poller the hook is a no-op and the error copy stays generic, which is why this id is in the portable bundle but not the runtime-safe preset and why the example above leaves it out.

`loop_detection` (display "Tool Loop Detection", no tools) appends a system warning to history when the model repeats itself, telling it to change approach. The detectors run most specific first.

- A mutating tool (`edit_file`, `write_file`, `delete_file`, `bash`, including namespaced `server__write_file`) failing identically twice in a row. The warning never echoes the untrusted error text.
- The same tool call producing the same result repeatedly.
- Re-reading the same file or output range while alternating with others.
- The same batch of tool calls with identical arguments, regardless of order.

Config keys are `threshold` (integer, minimum 1, default 3) for repeated batches, results, or read ranges, and `mutating_failure_threshold` (integer, minimum 1, default 2). Counts reset whenever a user or system message intervenes, and only one warning is injected per load.

`progress_guard` is runtime-enforced with no configuration; `ProgressGuardCapability::new()` builds it and every threshold is a fixed constant. It watches tool traffic in a coding agent and warns when investigation continues without progress, placing the warning inside the tool result under `progress_guard_warning`. Thresholds include 24 investigation tools without an edit or validation, a checkpoint demand at 48, the same target re-read 5 times, 3 consecutive zero-match searches, 3 repeated `git status` or `git diff` checks, re-running a validation such as `cargo test` on an unchanged workspace, and an edit that returns the workspace to a previously seen state. Polling an external event (`get_task`, `gh pr checks`, and similar) 4 times within the last 8 calls triggers a warning to run one detached `spawn_background` watch and end the turn, or to block on `wait_task`; both are background-task tools that delegate a long wait to a spawned task instead of polling from the main loop. When a read target returns byte-identical output of at least 512 bytes, the result is replaced by `{"unchanged_since_last_read": true, "tool": <name>}`. Any applied mutation resets the streaks, and there is no fixed session iteration cap.

## Safety: guardrails, prompt_canary_guardrail, tool_approval, error_disclosure

````rust
use serde_json::json;

let guardrails = CapabilityRef::new("guardrails").config(json!({
    "mode": "active",
    "checks": [
        {
            "id": "no-secrets-in-output",
            "stage": "output",
            "on_fail": "block",
            "replacement": "[Response withheld: appears to contain a credential.]",
            "type": "regex",
            "patterns": ["AKIA[0-9A-Z]{16}", "ghp_[A-Za-z0-9]{36}"]
        },
        {
            "id": "no-shell",
            "stage": "tool_use",
            "on_fail": "block",
            "type": "tool_pattern",
            "tools": ["bash*", "*exec*"]
        }
    ]
}));

let agent = Agent::builder()
    .instructions("Support agent for a public help desk.")
    .model(Model::simulated("Happy to help."))
    .capability(guardrails)
    .capability("prompt_canary_guardrail")
    .capability(CapabilityRef::new("error_disclosure").config(json!({ "mode": "generic" })))
    .build()?;
````

`guardrails` (display "Guardrails", category Safety, no tools, Risk Low) attaches declarative per-agent checks that inspect model output and tool activity and block or log on a match. With no checks configured the agent runs as before. The config is a top-level `mode` (`active` default, or `advisory`) and a `checks` array; each check has a `stage`, a `type`, an optional stable `id`, `on_fail` (`block` default, or `log`), an optional `replacement`, and type-specific fields. Stages are `output` (streamed assistant text), `tool_use` (tool name and arguments before execution), and `tool_output` (a result before it enters model context); `tool_output` is the trust boundary for untrusted content such as web pages and MCP responses.

- `regex`: `patterns` array, any stage.
- `blocklist`: `words` matched as substrings, `case_sensitive` default false, any stage.
- `tool_pattern`: `tools` array of names with `*` wildcards, `tool_use` only.
- `llm_judge`: a natural-language `prompt` evaluated by the utility LLM, `tool_use` and `tool_output`.
- `mcp`: delegation to an external guardrail with required `server` and `tool`, `tool_use` and `tool_output`.
- `moderation`: optional `categories` (default hate, harassment, self_harm, sexual, violence, illicit), `threshold` 0 to 100 default 50, `output` only, scored once on the finalized message.

A `block` on `output` or `tool_output` replaces the content with the check's `replacement` or a default notice, and the model's original tokens are never persisted; a `block` on `tool_use` rejects the call and feeds the reason back so the model can self-correct. `log` records the hit and continues, and `advisory` downgrades every hit to `log` so you can tune against false positives before enforcing. Every hit includes a stable `guardrail.<rule_type>` reason code such as `guardrail.regex`; localize copy from the code. Deterministic rules run in-process with no I/O and hard limits on count and length. Model-backed and MCP checks run on the async path, bounded by a 10 second timeout and at most 4 calls per invocation, and fail open on timeout or an unparseable verdict; `llm_judge` and `moderation` send excerpts to your own configured utility LLM. Invalid regexes and stage/type pairs are rejected at validation. You adopt a ready-made preset such as secret detection or dangerous-shell blocking by copying its `config` into the agent's `guardrails` config.

`prompt_canary_guardrail` (Safety, no tools, Risk Low) withholds an assistant message when the model echoes the first sentence of its system prompt. Nothing to configure: the canary is the first system-prompt sentence whose normalized form is at least 30 characters, capped at 240; short openers like "You are a helpful assistant." are skipped, and if no sentence qualifies the guardrail does not arm. Detection runs per text delta with lowercase and collapsed whitespace on both sides. On a hit the stream aborts and a canned refusal is persisted; the config key `replacement` customizes it, default "[Response withheld: the model attempted to reveal protected instructions.]". The event `output.message.replaced` includes `reason_code: "system_prompt_leak"`. The match is verbatim (a paraphrase does not trip it), first-sentence only, without partial matching, and on assistant text alone.

`tool_approval` (display "Tool Approval Gate", Safety, not auto-registered) suspends the turn and asks a human before a risky tool call executes. Construct it with `ToolApprovalCapability::new(Arc<dyn ToolApprover>)` and implement `approve(session_id, tool_call, tool_def)`, which blocks until a human answers `Allow`, `AllowAlways`, `Reject`, `RejectAlways`, `Cancelled`, or `Unavailable`. A host without an interactive prompt does not register the gate. Config key `mode` is `off`, `normal` (default; asks before tools declared destructive or outward-facing), or `protective` (asks before anything not declared read-only); risk comes from the tool's own `readonly`, `destructive`, and `open_world` hints, and an un-annotated tool counts as mutating. `AllowAlways` and `RejectAlways` are remembered per session and tool. A denial reaches the model as "Denied `<tool>` - <reason>.", and an unreachable approver blocks rather than failing open.

`error_disclosure` (display "Error Disclosure", no tools) controls how much of a run-blocking error reaches session viewers through config key `mode`: `generic` (one localizable message, for public-facing agents), `standard` (default even without the capability; stable code plus structured fields), or `detailed` (standard plus the driver error text, for trusted surfaces). A client may request a per-message mode through `controls.error_disclosure`, but it can only narrow disclosure, never widen it past the configured ceiling. The applied mode and pre-disclosure code are recorded in message metadata as `error_disclosure` and `source_error_code`.

## Tool discovery and tool-call behavior: auto, OpenAI, Claude, and client-side tool search, parallel_tool_calls, native_async_tools, tool_call_repair

````rust
use serde_json::json;

let agent = Agent::builder()
    .instructions("Operate the repository with whatever tools the task needs.")
    .model(Model::simulated("ready"))
    .capability(FileSystem)
    .capability(CapabilityRef::new("auto_tool_search").config(json!({ "threshold": 10 })))
    .capability(CapabilityRef::new("parallel_tool_calls").config(json!({ "mode": "prefer" })))
    .capability(CapabilityRef::new("tool_call_repair").config(json!({ "max_reprompts": 2 })))
    .build()?;
````

With many tools, sending every full parameter schema on every request consumes a large share of the context window. Tool search sends only names and descriptions and loads schemas on demand; a 19-tool surface measured about 67 percent smaller serialized tool lists.

`auto_tool_search` (Optimization; the recommended default and what Generic uses) picks the mechanism at runtime from the agent's model: OpenAI hosted search on GPT-5.4 and newer, Anthropic hosted search on Claude Opus 4, Sonnet 4.5 and 4.6, Haiku 4.5, and Fable 5 and newer, and the client-side `tool_search` everywhere else, including Gemini, gateway-served ids like `anthropic/claude-...`, and unknown models. `AutoToolSearchCapability::new()` uses the shared default threshold of 15 tools, `with_threshold(n)` forwards one threshold to all three mechanisms, and `with_never_defer([...])` forwards a `never_defer` list to the client-side fallback. In JSON the bare id enables the default and `{"capability_ref": "auto_tool_search", "config": {"threshold": 10}}` overrides it. Do not combine it with `openai_tool_search`, `claude_tool_search`, or `tool_search` on the same agent. For a bare first-party id served through an OpenAI-compatible gateway, the capability picks the hosted path and sends full schemas; add `tool_search` explicitly there.

`openai_tool_search` (Optimization, no tools) enables OpenAI's hosted deferred loading on `gpt-5.4*` and `gpt-5.5*`: deferrable tools get `defer_loading: true` and a `{"type": "tool_search"}` activator is appended. On any other model it is silently ignored with no fallback. `OpenAiToolSearchCapability::new()` uses threshold 15; `with_threshold(n)` or the JSON `threshold` key override it.

`claude_tool_search` (Optimization, no tools) enables Anthropic's hosted deferred loading: each deferrable tool gets `defer_loading: true` and a `tool_search_tool_bm25_20251119` server tool returns the 3 to 5 most relevant tools inline. Supported ids are `claude-opus-4*`, `claude-sonnet-4-5`, `claude-sonnet-4-6`, `claude-haiku-4-5`, `claude-fable-5-1`, and `claude-fable-5`; Bedrock and OpenRouter skip it silently. `ClaudeToolSearchCapability::new()` uses threshold 15.

`tool_search` (Optimization, client-side, 1 tool) works with any provider. Deferrable tools' schemas are replaced with a minimal stub and the model loads full schemas by calling `tool_search` with a keyword `query` such as "read file". The threshold is the total tool count at which deferral starts (default 15; `ToolSearchCapability::with_threshold(n)` or the `threshold` key change it), and `with_never_defer([...])` or the `never_defer` config array pins hot-path tools like `read_file`, `write_file`, `edit_file`, and `bash` so their full schemas stay loaded and the agent never needs a search round-trip before its first call. Per-tool `deferrable` policy is set by the tool's owner: `never` (full schema always, for high-frequency tools like `write_todos`), `automatic` (default), or `always`. At most 8 results return per call, revealed tools stay loaded for the session, and a miss returns `available_tools` so the model can refine. The `tool_search` tool itself is never deferred.

`parallel_tool_calls` (Optimization, no tools, Risk Low) controls whether the model is asked to emit several tool calls per turn and whether the scheduler runs a batch concurrently. Modes are `prefer` (default when enabled without a mode), `avoid` (one call per turn and forced serialization on every provider), and `none` (omit the preference, which neutralizes an inherited harness setting). OpenAI sends the top-level `parallel_tool_calls` boolean and Anthropic sends `tool_choice.disable_parallel_tool_use`; Gemini and Bedrock have no wire control, but the local scheduler still honors the preference.

`native_async_tools` lets a supported model keep reasoning while selected read-only lookup tools run natively in the background. It requires durable worker storage, meaning a persistent backing store that a hosted worker owns rather than the in-memory engine. Config key `tools` maps each tool name to `null` (plain function tool) or a custom tool-format object; a null or empty config enables `web_fetch` only.

`tool_call_repair` (display "Tool Call Repair", Safety, Risk Low, off unless enabled) repairs malformed tool-call arguments so the turn recovers instead of surfacing a raw parse error. Use it with models that wrap arguments in prose or code fences or emit lenient JSON. Salvage is a pure function over each call's arguments that extracts verbatim JSON without inventing fields: it unwraps fenced blocks, strips prose to the first balanced object, removes trailing commas, normalizes single quotes, and coerces strings to the schema's declared `integer`, `number`, or `boolean`. It runs in the core reason step after tool calls are finalized, so it applies to every driver. A call that still misses a `required` key goes to the bounded re-prompt path; `max_reprompts` (integer 0 to 5, default 1) caps corrective re-prompts per call. Each malformed call emits one `tool.call_repaired` event with an outcome of `local-salvage`, `re-prompt`, or `gave-up`.

## Context management: compaction, infinity_context, tool output persistence and distillation, prompt_caching

````rust
use serde_json::json;

let agent = Agent::builder()
    .instructions("Work through the whole backlog without losing the thread.")
    .model(Model::simulated("continuing"))
    .capability(FileSystem)
    .capability(CapabilityRef::new("compaction").config(json!({ "strategy": "auto", "proactive": true })))
    .capability(CapabilityRef::new("infinity_context").config(json!({ "context_budget_tokens": 80000, "min_recent_messages": 12 })))
    .capability("tool_output_persistence")
    .capability("tool_output_distillation")
    .capability("prompt_caching")
    .build()?;
````

`compaction` (display "Compaction", Optimization, no tools) compacts a conversation that would exceed the context window instead of failing, configured through JSON as above or through the typed `CompactionConfig`. `strategy` accepts `auto` (default; cascades observation masking, then the provider's native compact endpoint such as OpenAI responses compact, then summarization, then aggressive trim), `native` (the provider's native compaction operation only), `observation_masking` (replace older tool outputs with compact summaries), and `summarization` (ask the configured model to summarize older turns). `proactive` (boolean, default true) compacts before a provider rejects an oversized request, and `budget_percent` (number 0.1 to 1.0, default 0.85) sets that trigger: compaction runs once estimated tokens exceed the given fraction of the model's context window. Nested advanced objects stay out of the schema but are accepted.

- `observation_masking.keep_recent_tool_outputs` (default 2) keeps the newest N tool outputs verbatim and replaces older ones with summaries without changing the message count; `observation_masking.summary_format` is `one_line` (default, `[tool_name -> N lines, N bytes]`) or `head_tail`.
- `cost_control.enabled` (default true) masks stale bulky results on every call even when the window has room, so only the prompt view gets cheaper while storage stays lossless; its keys are `keep_recent_tool_results` (2), `mask_after_tool_results` (4), `max_live_tool_result_bytes` (24 KiB), and `compact_after_tool_result_bytes` (256 KiB).
- `summarization.model` (default none, the agent's model) can point at a cheaper model such as `claude-haiku-4-5-20251001`; `summarization.preserve` defaults to decisions, files_modified, errors, current_plan, and skill_instructions. The summary replaces the compacted prefix as a system message wrapped in `[CONVERSATION_SUMMARY] ... [/CONVERSATION_SUMMARY]`.
- `memory_tiers.hot_messages` (20, verbatim) and `memory_tiers.warm_messages` (100, masked); older messages are cold and summarized.

`activate_skill` results survive every stage and tool call and result pairs stay atomic; aggressive trim, the last stage, keeps the system prompt, the original task message, protected messages, and the newest messages that fit. Tokens are estimated as characters divided by 4, and compaction runs before infinity-context trimming (priority 50 versus 100).

`infinity_context` (display "Infinity Context", Optimization, 1 tool) keeps recent turns in the live prompt and trims older history out of it. The `query_history` tool (display "Query History") retrieves what was trimmed: `query` for case-insensitive keyword search, `message_range` with `from` (inclusive) and `to` (exclusive) zero-based indices, and `limit` (default 20, cap 50). Each record has index, position, role, created_at, and content truncated to 500 characters. Config keys are `context_budget_tokens` (default 100000), `min_recent_messages` (default 10, always kept), `max_recent_messages` (optional hard cap for public support chats), and `keep_first_messages` (default 0, maximum 16; pins the original task, and defaults to 0 so an untrusted oversized first message cannot bypass the budget). When messages are hidden the model sees `[IMPORTANT: N earlier messages are NOT visible in this context.]` between the anchored head and the recent window. With `compaction` also enabled, infinity context defers eviction so a summary rather than a bare notice covers old turns.

`tool_output_persistence` (display "Tool Output Persistence") saves full exec output to the session filesystem before truncation reaches the model. Output is written to `/outputs/{tool_call_id}.stdout` and `.stderr`, shown as `/workspace/outputs/...`. Only tools whose definition sets the `persist_output: true` hint are handled. When inline stdout is incomplete the result includes `full_output`, `total_lines`, and `output_files`, plus the annotation "[full output saved to <path> (<N> KiB) - use read_file with offset/limit]". The tool argument `output` selects `auto` (default), `normal`, or `full`, and each stream is capped at 1 MiB. Your own tool can call the `persist_output` helper (alias `persist_large_output`) and `compact_persisted_result_for_model`.

`tool_output_distillation` (display "Tool Output Distillation") covers tools that do not declare `persist_output`, in particular MCP tools and `web_fetch`, whose output was otherwise head-truncated by the 64 KiB hard limit. Results of at least 8 KiB are distilled by content shape: long strings clip to a head-plus-tail window of about 2 KiB, arrays keep 5 sample rows, unified diffs collapse to a diffstat, and small sibling fields stay intact, while the full original is persisted. The result includes `distilled: true`, `distill_note`, `full_output`, and `output_files`; error results are left verbatim.

`prompt_caching` (no tools, no prompt text) enables provider-specific prompt caching on outbound requests. `PromptCachingCapability::new()` selects the `Auto` strategy; `with_strategy(PromptCacheStrategy::Explicit)` caches only the developer-instruction prefix; `with_gemini_cached_content(strategy, "cachedContents/...")` points Gemini at a pre-created cache. Config keys `strategy` (`"auto"` or `"explicit"`) and `gemini_cached_content` override the constructor values, and the intent is recorded in `llm.generation` event metadata.

## Skills: the SKILL.md format, skills, skills_scoped, attach_skill, and the hello-world example

````markdown
---
name: hello-world
description: A simple example skill that demonstrates the Agent Skills format. Use this as a template for creating new skills.
license: MIT
metadata:
  category: examples
  version: "1.0"
---

Explain that you are using the "hello-world" skill.
````

````rust
use everruns::Skills;
use everruns::prelude::*;

let agent = Agent::builder()
    .instructions("Use a skill when one fits the task.")
    .model(Model::simulated("I'm using the hello-world skill right now."))
    .capability(FileSystem)
    .capability(Skills)
    .build()?;
````

The first block is the hello-world example skill. A skill is a portable instruction package in the agentskills.io format: a `SKILL.md` file with YAML frontmatter, opened and closed by a line of three hyphens, followed by a markdown body that becomes the instruction text the model receives on activation. It ships as a single markdown file or as a ZIP archive with scripts, references, and assets. The `version` in `metadata` is quoted so it stays a string.

- `name` (required): the kebab-case slug invoked as `/name`; 1 to 64 lowercase letters, digits, and hyphens, no leading, trailing, or consecutive hyphens.
- `description` (required): when to use it, up to 1024 characters.
- `license` and `compatibility`: informational, up to 500 characters each. `metadata`: a free-form map; `version` is read from it, default "1.0".
- `allowed-tools`: comma-separated tool patterns; omit to inherit the harness tool set.
- `user-invocable` (default true) and `disable-model-invocation` (default false); setting both, which hides the skill, produces a warning that it is unreachable. `argument-hint`: autocomplete text up to 128 characters.
- `context`: `inline` (default) or `fork` to run in an isolated subagent session. `agent`: the subagent type for fork, default `general-purpose`, an error without `context: fork`. `model`: a fork-only override, ignored inline with a warning.

The body is limited to 100 KB, and a body above 500 lines produces a warning. `$ARGUMENTS` expands to the whole argument string, `$ARGUMENTS[N]` and `$N` to the Nth zero-based argument, and a body with no placeholder gets an appended `ARGUMENTS: ...` line. `${SESSION_ID}` and `${SKILL_DIR}` resolve at activation. The backtick command-injection syntax is never expanded by the built-in tools. `parse_skill_md` returns structured data or every validation error; `validate_skill_md` returns a report with `valid`, `name`, `description`, `errors`, and `warnings`.

The second block attaches `skills` (Core, 2 tools) with the zero-config `Skills` marker; the bare id works too, and there are no tunable options. The capability ships with no skills of its own, so you upload `SKILL.md` packages to `/.agents/skills/{name}/SKILL.md`, shown to the agent as `/workspace/.agents/skills/...`, and the agent discovers them at runtime. `list_skills` (display "List Skills", no parameters) returns a `skills` array with name, description, path, version, `user_invocable`, and `disable_model_invocation`, plus `count` and `skills_path`; malformed skills include an `error`. `activate_skill` (display "Activate Skill") takes the required `name` and optional `arguments` and returns the instructions wrapped in `<skill name="...">`; fork skills add `context: "fork"`, `agent`, and optional `model`. Activating the same skill again returns the cached result with `already_active: true`. Discovery costs about 100 tokens per skill and activation loads under 5000. The system prompt lists up to 15 skills each turn, marking user-invocable ones `(/name)` and omitting model-hidden ones. Names containing `..`, `/`, or `\` are rejected before any file access.

`ScopedSkillsCapability::new(SkillsConfig { scopes, resolver, manage_tools })` is a drop-in replacement under the same id `skills` that reads skills from several labeled sources at once. Each `SkillScope::new(label, vfs_root, writable)` has a stable label shown to the model (`workspace`, `global`, `system`), a VFS root, and a writable flag; earlier scopes win on name conflicts. `SkillsConfig::default()` reproduces the plain capability with one writable `workspace` scope at `/.agents/skills`. Implement `SkillDirResolver` to control where `${SKILL_DIR}` resolves, for example a CLI whose shell runs on the host. Set `manage_tools: true` to expose `read_skill` (`name`, optional `scope`; returns raw `skill_md` and a manifest of bundled files) and `write_skill` (`name`, `skill_md`, optional `scope`, `files` map of relative path to content, `overwrite` default true; the frontmatter name must match, and writes to read-only scopes are rejected). Files are capped at 1 MiB and 64 per skill.

`AttachSkillCapability::from_registry(skill_id, name, description, instructions, files)` mounts a database-registered skill into the session as a read-only directory at `/.agents/skills/{name}/`, a mount point layered into the session filesystem, so the skills tools discover it alongside uploaded skills. Bundled `(path, content)` pairs such as `scripts/run.sh` sit next to the regenerated `SKILL.md`, and `from_registry_with_options` sets `user_invocable` and `disable_model_invocation`. Each attached skill has the capability id `skill:{uuid}` and is opt-in rather than part of the auto-registered bundle. Code-defined capabilities can bundle skills through `contribute_skills()`, which become read-only mounts at the same path.

## Task lists, system commands, and subagent delegation: stateless_todo_list, system_commands, spawn_agent

````rust
use everruns::StatelessTodoList;
use everruns::prelude::*;

let agent = Agent::builder()
    .instructions("Plan the migration as a task list, then work through it.")
    .model(Model::simulated("Tracking three tasks."))
    .capability(StatelessTodoList)
    .build()?;
````

`stateless_todo_list` (display "Task Management", 1 tool) gives the agent a structured task list for multi-step work, attached with the zero-config `StatelessTodoList` marker or the bare id. The `write_todos` tool (display "Write Todos") takes a `todos` array and replaces the whole list on each call. Each task has `content` (imperative), `activeForm` (present-continuous, for live display), and `status` (`pending`, `in_progress`, or `completed`); a missing or invalid field returns a per-task error with a 1-based index. The result echoes the list with `total_tasks`, `pending`, `in_progress`, and `completed` counts and a soft `warning` when no task or more than one is in progress; an empty array clears the list. The list lives only in conversation history. The prompt section tells the model to use it sparingly and to keep exactly one task `in_progress`, marking `completed` only when fully done; a greeting or a single-step edit does not warrant a list, and neither does a read-only check.

`system_commands` is a built-in slot for `/slash` commands that execute directly without a round-trip to the LLM.

`subagents` (Core, feature `subagents`, 3 tools) is a hosted Platform capability. It lets an agent delegate work to subagents that run in their own isolated context windows so verbose or independent tasks do not clutter the main conversation. Subagents inherit the parent's harness and agent configuration but keep their own message history, and they spend from the root session's budget pool. The unified `spawn_agent` tool requires `name` (unique within the session), `instructions` (the initial prompt), and `target`, where `target.type` is `subagent`, `agent`, or `external_a2a`; the last reaches a remote agent and requires the separate `a2a` opt-in. `mode` is `background` (default; returns a `task_id`, an identifier for work that continues after the tool returns, and notifies the session on finish, capped at 6 hours) or `foreground` (waits and returns inline, 5 minute timeout). `lifetime` is `linked` (default) or `detached`. `seed` is `fresh` (default), `fork`, or `workspace`, and selects the starting state the child begins from. `blueprint` names a specialist such as `github_scout` with its own prompt, model, and private tools, with `config` alongside; `result_schema` and `message_schema` shape the exchange; `wait_timeout_secs` (1 to 86400) applies to external A2A only. Blueprint and config are valid only for subagent targets, and detached lifetime and `message_schema` are not valid for external A2A.

A spawned subagent is monitored and steered through the session task tools by `task_id`: `list_tasks` with `kind: "subagent"`, `get_task`, `message_task`, `cancel_task`, and `wait_task`. Nesting is bounded by the `subagents` config keys `max_subagent_depth` (default 2; 0 blocks spawning), `max_active_descendant_tasks` (16), `max_total_descendant_tasks` (200), plus `max_active_detached_tasks` and `max_total_detached_tasks`; the host reads these into a nesting policy that caps every descendant of a session.

The Framework-level pattern without the hosted capability is the subagents example:

````rust
let coordinator = Agent::builder()
    .name("coordinator")
    .instructions("Delegate five independent reviews, then wait for and summarize all results.")
    .provider(OpenAI::from_env()?)
    .model(MODEL_ID)
    .tool(spawn_tool(/* application-owned task registry */))
    .tool(wait_tool(/* the same registry */))
    .build()?;
````

Each child is an ordinary agent built inside `tokio::spawn` with `Agent::builder().name(format!("worker-{}", index + 1)).instructions(...).provider(OpenAI::from_env()?).model(MODEL_ID).build()` and run through its own `Engine::new().create(child).run(instruction).await`, returning `result.response`. The application owns a task registry, a map from task ids to running child work that the spawn tool inserts into and the wait tool reads, so the coordinator sees the same spawn-then-wait pattern as the hosted tool. Run it with cargo for package `everruns`, feature `openai`, example `subagents`; it requires the `openai` feature.

## User hooks and hook bundles

````json
{
  "hooks": [
    {
      "id": "guard_rm",
      "event": "pre_tool_use",
      "matcher": {
        "tool_name": "bash",
        "args_jsonpath": "$.command",
        "deny_regex": "(?:^|;|&&|\\|)\\s*rm\\s+-rf\\b"
      },
      "executor": {
        "type": "bash",
        "command": "echo '{\"decision\":\"block\",\"reason\":\"rm -rf is blocked by policy\",\"user_message\":\"That command is blocked by policy.\"}'"
      },
      "on_error": "block",
      "description": "Reject any bash invocation that pipes, chains, or starts with rm -rf"
    }
  ]
}
````

This is the shape of `block-rm-rf.json`, one of the ready-to-paste bundles in examples/hook-bundles: a JSON file with a top-level `hooks` array whose entries go into the `user_hooks` capability config. `user_hooks` (category Automation, Risk High, no tools) is a hosted Platform capability that runs user-authored shell commands at six lifecycle events: `session_start`, `user_prompt_submit`, `pre_tool_use`, `post_tool_use`, `turn_end`, and `session_end`. A hook can audit silently or mutate the inputs it observes, and at `user_prompt_submit` and `pre_tool_use` it can also block the action; a blocked prompt fails the turn with the hook's `user_message`. Mutation surfaces are the user prompt text, `ToolCall.arguments` before execution, and `ToolResult` after.

- `id` (optional, defaults to `{event}_{idx}`) and `event`.
- `matcher`, tool events only: `tool_name` exact, `tool_name_glob` with `a|b|c` alternation or a trailing `*`, and `args_jsonpath` (a dot-path such as `$.command` into the arguments) with a mutually exclusive `match_regex` or `deny_regex`. Regexes use the Rust regex flavor, with no look-around or backreferences.
- `executor`: `{"type": "bash", "command": ..., "env": {...}}`.
- `timeout_ms` (100 to 30000, default 5000).
- `on_error`: `block`, `allow`, or `warn` (default); governs executor failures such as a timeout or non-JSON output.
- `description`.

The script receives its payload through environment variables, not stdin: `EVERRUNS_HOOK_PAYLOAD_JSON`, a file copy at `EVERRUNS_HOOK_PAYLOAD_PATH`, and the scalars `EVERRUNS_HOOK_EVENT`, `EVERRUNS_HOOK_ID`, `EVERRUNS_HOOK_SESSION_ID`, `EVERRUNS_HOOK_TURN_ID`, `EVERRUNS_HOOK_TOOL_NAME`, and `EVERRUNS_HOOK_TOOL_CALL_ID`. The envelope has `event`, `hook_id`, `session_id`, `turn_id`, `org_id`, `agent_id`, `ts`, and an event-specific `data`; for `pre_tool_use` that is `tool_name`, `tool_call_id`, and `arguments`. The script answers on stdout with `{"decision": "allow" | "mutate" | "block", "reason": ..., "user_message": ..., "patch": {...}}` or `{}`. The git-hook convention also works: empty stdout with exit 0 means allow, and a non-zero exit means block, with stderr as the reason. A `user_prompt_submit` hook rewrites the prompt with a `mutate` decision and `patch.message`, for example `jq -c '{decision:"mutate",patch:{message:("[reminder: follow the house style guide]\n" + .data.message)}}'`.

Hooks run inside the sandboxed shell against the session filesystem with the session's egress policy and no host shell. The default timeout is 5 seconds with a 30 second maximum, and combined output is capped at 64 KiB. Hooks chain in capability-declaration order and then array order; the first `block` wins and earlier mutations survive. Custom capabilities ship hook specs by overriding `Capability::user_hooks_with_config`.

The other bundles cover the remaining events.

- `format-on-edit.json`: `post_tool_use` on `edit_file`, hooks `fmt_rs` and `fmt_ts_js` matching `.rs` and web file paths, running cargo fmt or prettier with `on_error: warn`.
- `audit-every-tool.json`: matcher-less `post_tool_use` appending `tool_name` to `/workspace/.audit.log`.
- `session-bootstrap.json`: `session_start` writing `/workspace/.bootstrap.md`.
- `block-secret-prompt.json`: `user_prompt_submit` blocking pasted private keys, `on_error: block`.
- `turn-end-log.json`: `turn_end` appending `ts`, `turn_id`, and `data.success` to `/workspace/.turn-log`.

*2026-09-17 03:24 - claude-fable-5.1*
