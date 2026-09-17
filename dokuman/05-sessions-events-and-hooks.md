<!-- source: everruns/everruns @ 6cf2c15e5 (crate everruns 0.20.0). The working tree fast-forwarded to 7d9e07a0b (0.21.1) during extraction; identifiers and examples were audited against 6cf2c15e5. -->
# Sessions, Events, and Hooks

A built agent does nothing until an engine creates a session from it, and everything interesting about running an agent happens inside that session. Each message you send becomes one turn through the Input, Reason, Act loop, and every step of that turn is published as a typed event you can render live. Every boundary the loop crosses is also a place where your own code can run before execution continues.

## Sessions, turns, and messages: create, resume, send, steer, wait

### Create, run, and resume

````rust
use everruns::{Agent, Engine, Model, SessionId};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let engine = Engine::new();

    let session_id: SessionId = {
        let agent = Agent::builder()
            .instructions("Answer briefly.")
            .model(Model::simulated("noted"))
            .build()?;
        let session = engine.create(agent);
        let first = session.send_and_wait("My project is Atlas.").await?;
        assert!(first.success);
        session.session_id()
    };

    let resumed = engine.resume(session_id).await?;
    let second = resumed.send_and_wait("Continue with that project.").await?;
    println!("{}", second.response);
    Ok(())
}
````

`engine.create(agent)` takes ownership of the agent and returns a live `Session`. The agent value goes out of scope at the end of the block, the `Session` handle is dropped with it, and the conversation survives both, because the engine snapshots the immutable agent at creation and keeps the session in its catalog. `session.session_id()` returns a typed `SessionId` that is copyable and printable; keep it when the application may need to reopen the conversation later. `session.id()` returns the same identity as a plain `String` for lining up log lines and carries no organization or principal identity.

`engine.resume(session_id)` returns the same live session if a handle is still open, and otherwise reopens the persisted environment and constructs a fresh handle. Either way the resumed session continues the conversation with the earlier turns in its history, and resuming a session that was created but never run succeeds with an empty history. Failures come back as `ResumeError`: `SessionNotFound { session_id }` (also the result when the id came from a different engine), `Unavailable`, `Corrupt`, `WorkspaceProviderUnavailable { provider_id }`, `WorkspaceUnavailable`, `WorkspaceMismatch`, and `WorkspaceBindingCorrupt`. Backend details stay private.

Two sessions created from the same agent never share history; they have different ids and each history grows only from its own turns. Several sessions on one engine can run at once with an ordinary `tokio::join!`, and the responses do not cross. `Engine` and `Session` are cheap to clone: engine clones share one session catalog, and session handle clones share the same conversation.

### Send, steer, and wait

`send_and_wait` and `run` both drive one turn to completion and return a `Turn`. A turn that ends in a refusal or at the iteration ceiling still comes back as `Ok(Turn)` with `success == false`; only a runtime error that produces no turn at all is `Err`. `send` is the non-blocking path.

````rust
use everruns::prelude::*;

let initial = session.send("Plan a three-day trip to Lisbon.").await?;
let latest = session.send("Prefer trains over flights.").await?;

match latest.disposition {
    SendDisposition::Steered => println!("joined running turn {}", latest.turn_id),
    SendDisposition::Started => println!("first turn had finished; started {}", latest.turn_id),
    _ => {}
}

let turn = latest.wait().await?;
println!("{}", turn.response);
````

`send` returns as soon as the message is accepted, not when the model answers. The receipt is a `SentMessage` with `message_id`, `turn_id`, and `disposition`. A message sent while a turn is active steers that turn: it joins at the turn's next reason boundary, the receipt reports `SendDisposition::Steered`, and `turn_id` matches the running turn. After a turn finishes, a fresh `send` starts a new turn and reports `SendDisposition::Started`. The enum is non-exhaustive; keep the wildcard arm. `wait()` on the receipt yields the turn that accepted the message in either case, and `receipt.turn()` returns a cloneable `TurnHandle` whose `id()` is shared with events and the final `Turn`.

Every send and run method takes `impl Into<InputMessage>`, which accepts a `&str` or a rich message built from `ContentPart` values such as `ContentPart::text(..)` and `ContentPart::image_url(url)`. User-submitted input is restricted to text, image, image file, and file parts; tool calls and tool results are system-generated. Errors from these methods form the closed `RunError` set: `Runtime`, `Hook(HookFailure)`, `Environment`, `SessionClosed`, and `SteeringQueueFull`, the last meaning the active turn already holds the maximum number of pending steering messages.

Before the first turn or between turns, `session.inspect().await?` returns a `SessionContext` with `model`, `messages`, `tools`, `instructions`, `locale`, and `plugin_warnings`: the exact context the next model call will receive, assembled through the same path execution uses, including MCP tool discovery and plugin prompt contributions (chapters 06 and 10), with no lifecycle handler run. Each `ToolInfo` carries the model-facing `name` and `description` plus its `parameters` JSON Schema. Use it for assertions and debugging.

## The turn record, stop reasons, and completion gating

````rust
let turn = session.run("Summarize the plan.").await?;
println!(
    "turn {} stopped with {:?} after {} iterations and {} tool calls",
    turn.turn_id, turn.stop_reason, turn.iterations, turn.tool_calls
);
if !turn.success {
    eprintln!("error: {}", turn.error.clone().unwrap_or_default());
}
for failure in &turn.hook_failures {
    eprintln!("hook warning: {failure}");
}
````

`Turn` is a small, stable projection of the runtime's result: `response`, `turn_id`, `stop_reason`, `iterations`, `tool_calls`, `success`, `error`, and `hook_failures`. It contains no stores, session records, or platform identity, and it is non-exhaustive, leaving room for new fields.

`stop_reason` is a `TurnStopReason`, one provider-neutral value rather than a vendor finish string. The variants are `EndTurn`, `MaxTokens`, `MaxTurnRequests`, `Refusal`, `Error`, and `Cancelled`, serialized on the wire as `end_turn`, `max_tokens`, `max_turn_requests`, `refusal`, `error`, and `cancelled`. Provider finish reasons are normalized at the runtime boundary by `TurnStopReason::from_provider_finish_reason`, which matches case-insensitively.

- `length`, `max_tokens`, `max_output_tokens` become `MaxTokens`
- `refusal`, `content_filter`, `safety` become `Refusal`
- `error` becomes `Error`
- `cancelled` or `canceled` become `Cancelled`
- anything unrecognized, including `None`, becomes `EndTurn`

`MaxTurnRequests` is never produced from a provider string. The engine sets it when its own iteration cap cuts off pending tool calls or steering messages. On a failed model call a refusal stays distinct from a generic error. When the loop hits the cap, the error reads "Max iterations (N) reached" and the user-facing payload carries the code `max_iterations` with the configured maximum.

An application that auto-continues an agent needs to decide whether the user's request is finished. `gate_turn`, defined in the `everruns-core` turn completion module, is a pure function over what the turn already reported. The gate types and `ContinuationBudget` are not re-exported by the `everruns` crate, so this example imports them from `everruns_core::turn_completion` directly.

````rust
use everruns_core::turn_completion::{CompletionState, GateDecision, TurnSummary, gate_turn};

let summary = TurnSummary {
    success: turn.success,
    stop_reason: turn.stop_reason,
    response: &turn.response,
    tool_calls_count: turn.tool_calls,
    has_active_background: false,
};
match gate_turn(&summary) {
    GateDecision::Conclusive(CompletionState::Achieved) => println!("done"),
    GateDecision::Conclusive(state) => println!("stopped: {state:?}"),
    GateDecision::Evaluate => println!("tool-using work needs a semantic check"),
}
````

`CompletionState` has five values, `Achieved`, `Blocked`, `Failed`, `WaitingOnBackground`, and `InProgress`, and the gate rules run in order. An unsuccessful turn is `Failed`, except a cancelled one is `Blocked`, since only the user can clear it. `Error` or `Refusal` stop reasons are `Failed`. `Cancelled` is `Blocked` regardless of text. `MaxTokens` or `MaxTurnRequests` mean `InProgress`, stopped mid-task. Active background work with at least one tool call is `WaitingOnBackground`. An empty or whitespace-only response is `InProgress`. A response that asks the user something is `Blocked`. Otherwise a turn with no tool calls is `Achieved`, and a turn with tool calls returns `Evaluate` because mutations and multi-step work are ambiguous enough to justify a semantic check. The question heuristic is conservative: it requires a trailing question mark plus one of the markers `need`, `which`, `what`, `could you`, or `please provide`, so a reply like "shall I continue? I'll start with the parser." does not strand the user.

A runaway loop is caught whichever resource it leaks, because `ContinuationBudget` bounds auto-continuation for one user request in turns and tokens and also in wall-clock time. `ContinuationBudget::default()` allows 6 turns, 64,000 tokens, and 10 minutes; `ContinuationBudget::new(max_turns, max_tokens, max_elapsed)` sets custom limits. `observe_turn(tokens)` records a completed turn and returns `false` the moment any limit is crossed. The recorded turn counts first, which makes the budget a ceiling on completed work, not on attempted work. A zero-turn or zero-token budget disables continuation entirely. `reset()` clears usage and keeps the limits, and `usage()` returns the `(turns, tokens)` consumed so far.

## Session history, pagination, and derivation from events

````rust
let page = session.history().page().await?;
for message in &page.messages {
    println!("{}: {}", message.role, message.text());
}
if let Some(cursor) = page.next_cursor {
    let more = session.history().limit(25)?.after(cursor)?.page().await?;
    println!("{} more messages", more.len());
}
````

`session.history()` returns a `HistoryQuery`, and `page().await` is the happy path: at most 100 messages in canonical event-sequence order plus an opaque `next_cursor` when more remain. A brand-new session yields an empty page with no cursor, and a page that ends on the last message in history has no cursor either; there is never a phantom trailing empty page. `HistoryPage` exposes `session_id`, `messages`, `len()`, `is_empty()`, and `next_cursor`. Each `SessionMessage` is a transcript-only projection with `id`, `role`, `content`, and `created_at`. Its `text()` joins only the text parts, while images, tool calls, and tool results stay in `content`. Ordering is by event sequence, never by timestamp. Each completed turn appends exactly two messages, the user input then the agent reply, and steering messages sent mid-turn are retained in arrival order.

`limit(n)` raises the page size to at most 256 and validates eagerly: zero and anything above 256 return `HistoryError::InvalidLimit { requested, maximum }`, which carries the allowed maximum and leaves the application no need for a backend constant. `after(cursor)` continues where the previous page left off. For a full walk, `pages()` returns a fused `HistoryPages` reader.

````rust
let mut pages = session.history().limit(50)?.pages();
while let Some(page) = pages.next_page().await? {
    for message in page.messages {
        println!("{}", message.text());
    }
}
````

Each call reads at most one page and retains none of the earlier ones. After the final page, `next_page()` keeps returning `Ok(None)`, and a brand-new session yields a single empty terminal page.

A cursor pins a consistent snapshot. The first page fixes an event high-water mark, and continuations cannot see messages appended concurrently, so they neither skip nor duplicate. Start a new query to observe events committed after the snapshot. `HistoryCursor` is opaque and session-bound, and it is safe to persist: `to_string()` serializes it and `HistoryCursor::from_str` restores it, with `HistoryCursorParseError` for tokens that are malformed, over 4096 bytes, or carrying delimiter characters. The parse error carries no copy of the rejected token, and the cursor's `Debug` output redacts it. A cursor issued by a different session is rejected with `HistoryError::CrossSessionCursor`. The remaining `HistoryError` variants are `InvalidCursor`, `ExpiredCursor` (the backend no longer retains the snapshot), `IncompatibleCursor` (a different query or framework version), `HistoryTooLarge` (the bounded raw-event replay limit was reached before a page boundary was found), `SessionNotFound { session_id }`, `Unavailable`, and `Corrupt`.

History is derived from canonical persisted events rather than from a separate writable message store. The transcript is rebuilt in persisted sequence order from `input.message`, `output.message.completed`, and relevant `tool.completed` events, and ephemeral streaming deltas are excluded by design. An `output.message.replaced` event alone creates no history message; the subsequent completed message carries the safe replacement. The projection behaves the same way against the in-memory default engine as it will against durable persistence (chapter 08).

## Canonical events, live subscription, and cancellation

### Subscribing

````rust
use everruns::prelude::*;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let agent = Agent::builder()
        .instructions("Be brief.")
        .model(Model::simulated("Hello, world!"))
        .build()?;
    let session = Engine::new().create(agent);

    let mut events = session.events();
    let pending = session.send("hi").await?;

    while let Some(event) = events.recv().await? {
        match &event.kind {
            SessionEventKind::OutputStarted { .. } => print!("assistant: "),
            SessionEventKind::TextDelta { delta } => print!("{delta}"),
            SessionEventKind::ToolStarted { tool_name, .. } => eprintln!("starting {tool_name}"),
            SessionEventKind::ToolCompleted { tool_name, success, .. } => eprintln!("{tool_name}: {success}"),
            SessionEventKind::TurnFailed { error } => eprintln!("turn failed: {error}"),
            _ => {}
        }
        if event.kind.is_terminal() {
            break;
        }
    }
    let turn = pending.wait().await?;
    println!("\n{} iterations", turn.iterations);
    Ok(())
}
````

Call `session.events()` before `send`; the feed observes only events emitted after it is created, and several subscribers may attach to one session. `recv()` yields the next event. It returns `Ok(None)` once the session is dropped and nothing further can arrive, and it returns `EventStreamError::Lagged { missed }` when the consumer fell behind. `is_terminal()` is true for `TurnCompleted`, `TurnFailed { .. }`, and `TurnCancelled`, and `try_recv()` returns already-buffered events without waiting. The stream is bounded at `EVENT_STREAM_CAPACITY` (4096 events) and never applies backpressure to the turn: on overflow it evicts the oldest unread events and reports the exact gap, and the next `recv()` continues from the oldest retained event. Dropping the session closes the stream after buffered events drain, without deleting the session from the engine. The feed is not a durable replay API; after lag or a restart, rebuild from history pages.

### The typed kinds and the two surfaces

`SessionEventKind` has twenty variants and is non-exhaustive.

- Turn lifecycle: `InputMessage { message_id }`, `TurnStarted`, `TurnCompleted`, `TurnFailed { error }`, `TurnCancelled`
- Output streaming: `OutputStarted { message_id }`, `TextDelta { delta }`, `OutputReplaced { message_id, replacement }`, `OutputCompleted { message_id }`
- Tool lifecycle: `ToolStarted { tool_call_id, tool_name }`, `ToolCompleted { tool_call_id, tool_name, success }`, `ToolProgress { tool_call_id, tool_name, message }`, `ToolOutputDelta { tool_call_id, tool_name, stream, delta }`
- Inference steps: `ReasonStarted`, `ReasonCompleted { success, error }`
- Model reasoning: `ReasoningDelta { delta, accumulated }`, `ReasoningCompleted { text }`, `ReasoningItem { provider, item_id, summary }`
- Accounting: `ModelGeneration { model, provider, input_tokens, output_tokens, cost_usd, duration_ms, success }`
- Fallback: `Other { event_type }`

Concatenating `TextDelta` values reconstructs the response exactly, and one `message_id` spans `OutputStarted`, the deltas, and `OutputCompleted`. Deltas are provisional; completed events are authoritative, and `OutputReplaced` means discard everything accumulated for that message and show `replacement`. Model reasoning arrives on its own channel and must never be rendered as assistant text, while `ReasonStarted` and `ReasonCompleted` bracket each inference call of the reason/act loop and are not reasoning content. Unknown types arrive as `Other` with the stable dot-notation string; match it with a wildcard arm.

Every event exposes `event_id`, `session_id`, and optional `turn_id` as strings, plus `event_type()` (for example `turn.started`), `timestamp()` in RFC 3339, and `sequence()`, which is `Some` for durable events and `None` for ephemeral ones such as deltas. Of the two data surfaces, `data()` and `as_json()` are the reviewed surface, carrying only promoted fields, so they are safe to log or forward. `canonical_json()` is the complete envelope for auditing and replay, including prompts, tool arguments, and tool results, and it can gain fields in a patch release. Arguments and results stay off the reviewed surface because a tool call can carry credentials in and file contents out. The protocol exports its dot-notation type strings as constants such as `INPUT_MESSAGE`, `TURN_SEALED`, and `LLM_GENERATION`.

### Cancellation and timeouts

````rust
use std::time::Duration;

let cancel = CancellationToken::new();
let options = RunOptions::new()
    .cancel_token(cancel.clone())
    .timeout(Duration::from_secs(30));
tokio::spawn(async move {
    tokio::time::sleep(Duration::from_millis(200)).await;
    cancel.cancel();
});
let stopped = session.run_with("Start a long task.", options).await?;
assert!(!stopped.success);
assert_eq!(stopped.stop_reason, TurnStopReason::Cancelled);
````

Cancellation is cooperative and drop-based. Cancelling the token drops the turn's future at the next await point, and `run_with` resolves to `Ok(Turn)` with `success == false` and `stop_reason == TurnStopReason::Cancelled`. Cancellation is an outcome, not an error. Every clone of a token shares one signal, `cancel()` is idempotent, and `is_cancelled()` reports state. A token cancelled before the call stops the turn before it starts and skips every lifecycle hook. `timeout` shares the same semantics: on expiry the in-process turn is stopped and the cancelled turn is returned. With neither set, `run_with` behaves identically to `send_and_wait`. A specific active turn can also be cancelled from its handle with `pending.turn().cancel().await?`, which fails with `CancelError::SessionClosed` or `CancelError::TurnFinished` when it cannot apply. After cancellation a durable `turn.cancelled` event arrives with reason "cancelled by application". The `observe_and_cancel` example (requires the `openai` feature) shows both styles.

## Lifecycle hooks: agent start, turn start, tool start, tool end, completion, and user hooks

### The five points

````rust
use everruns::prelude::*;
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};

/// Report the current status of a named service.
#[everruns::tool]
async fn service_status(service: String) -> Result<String, String> {
    Ok(format!("{service}: operational"))
}

let tool_calls = Arc::new(AtomicUsize::new(0));
let counter = tool_calls.clone();

let agent = Agent::builder()
    .instructions("Always call service_status before answering a status question.")
    .provider(OpenAI::from_env()?)
    .model("gpt-5.6-terra")
    .tool(service_status())
    .on_agent_start(|context| async move {
        println!("agent {} started session {}", context.agent_name, context.session_id);
    })
    .on_turn_start(|context| async move {
        if context.input.content.is_empty() { Err("empty input") } else { Ok(()) }
    })
    .on_tool_start(|context| async move {
        println!("calling {} with {}", context.tool_name, context.arguments);
    })
    .on_tool_end(move |context| {
        let counter = counter.clone();
        async move {
            counter.fetch_add(1, Ordering::Relaxed);
            println!("tool {} finished (success={})", context.tool_name, context.success());
        }
    })
    .on_completion(|context| async move {
        println!("turn completed: {:?}", context.turn.stop_reason);
    })
    .build()?;
````

Hooks are awaited extension points attached to the agent, registered on `Agent::builder()` alongside `name`, `instructions`, `provider`, `model`, and `tool`. Register a hook when work must finish before execution continues; use `session.events()` when code only needs a non-blocking observation stream. Handlers are async `Fn` closures. An infallible handler returns `()`, and a fallible one returns `Result<(), E>` for any `E: Display`. Capture shared state through `Arc` values as the tool-end handler above does. The framework awaits each per-point chain in registration order. Contexts are owned snapshots: hooks cannot rewrite prompts, arguments, results, or outcomes. Because every session shares the same handlers, separate sessions and parallel tool calls may invoke a handler concurrently; protect shared mutable state inside the closure. Panics are not caught, and no hook timeout is applied; external work needs an application timeout inside the handler. The five points print as `agent_start`, `turn_start`, `tool_start`, `tool_end`, and `completion`, matching the `HookPoint` variants `AgentStart`, `TurnStart`, `ToolStart`, `ToolEnd`, and `Completion`.

- Agent start runs once per session just before its first turn with `AgentStartContext { agent_name, session_id }`. An error stops the turn, and the next run retries the whole chain, which is why handlers with external effects should be idempotent. Inspecting a session does not invoke it.
- Turn start runs before every turn with `TurnStartContext { agent_name, session_id, input }`. It sees the exact input but cannot change it, and the first error prevents the turn; the example rejects empty input.
- Tool start runs before every model-requested tool call with `ToolStartContext { session_id, turn_id, tool_call_id, tool_name, arguments }`. An error blocks only that call and skips later start handlers for it. The model sees the generic text "tool call blocked by tool_start hook #N" so credentials or backend diagnostics never reach it; the detailed message stays application-facing. It runs after host-configured execution gates such as the `tool_approval` builtin (chapter 07).
- Tool end runs after every call settles, including calls blocked by an earlier gate, with `ToolEndContext` carrying `arguments`, `result`, `error`, and a `success()` accessor. Errors cannot undo the call, and all handlers still run.
- Completion runs after a non-cancelled turn reaches a terminal outcome with `CompletionContext { agent_name, session_id, turn }`, whether `turn.success` is true or false. Once completion begins, cancelling the run token does not interrupt the chain.

Agent-start and turn-start errors return `RunError::Hook(failure)` to the caller. A `HookFailure` carries `point`, a zero-based `handler_index` in registration order, `message`, and for tool points `tool_name` and `tool_call_id`; it displays as "tool_start hook #1 failed: ...". Tool-start, tool-end, and completion failures never change `success`; they appear on `turn.hook_failures`, scoped to one turn. Around a single tool call the order is start handlers, the tool, end handlers, then completion. Cancellation during agent start, turn start, or an in-flight tool chain drops the active handler future, skips the remaining work, and does not run completion. The `lifecycle_hooks` example registers all five points around this same tool and requires the `macros` and `openai` features.

### User-defined hooks

Separately from in-process hooks, user-defined hooks bind shell-script handlers to lifecycle events through declarative `UserHookSpec` data (chapters 06 and 07): `user_prompt_submit` and `turn_end` for the turn lifecycle, `session_start` and `session_end` for the session lifecycle. A `user_prompt_submit` hook prints a JSON decision on stdout. `{"decision":"block","reason":"nope","user_message":"blocked"}` aborts the turn with the reason logged and audited; `{"decision":"mutate","patch":{"message":"rewritten"}}` replaces the prompt text. Prompt hooks chain in declaration order, each seeing the previous rewrite, and the first block stops the chain. A rewritten prompt applies only to the assembled provider context; persisted history keeps the original. Each spec sets an on-error policy for executor failures, `Block`, `Warn`, or `Allow`, but a non-zero exit always blocks. Session start, session end, and turn end hooks are advisory: a failure, block, or mutate never alters the session or turn. Hook output is capped at 64 KiB, and an invalid spec is dropped with a warning while the rest keep running. When a `user_prompt_submit` hook is configured the host fails closed by removing the `query_history` tool; otherwise filtered text could leak through history queries.

## Tool hooks (pre and post) and output hooks

````rust
use async_trait::async_trait;
use everruns_core::ToolContext;
use everruns_core::tool_hooks::{PreToolUseDecision, PreToolUseHook};
use everruns_provider::tool_types::{ToolCall, ToolDefinition};

struct DenyDeletes;

#[async_trait]
impl PreToolUseHook for DenyDeletes {
    async fn before_exec(
        &self,
        tool_call: ToolCall,
        _tool_def: &ToolDefinition,
        _context: &ToolContext,
    ) -> PreToolUseDecision {
        if tool_call.name == "delete_file" {
            PreToolUseDecision::Block {
                tool_call,
                reason: "deletes are disabled in this deployment".to_string(),
                user_message: None,
            }
        } else {
            PreToolUseDecision::Continue(tool_call)
        }
    }
}
````

A capability contributes pre-tool-use hooks by returning them from `pre_tool_use_hooks()` or `pre_tool_use_hooks_with_config()`, and they run before each individual tool call for every tool the agent calls, including built-in, MCP, and client-side tools that other sources contributed. Each hook receives the call by value and returns `PreToolUseDecision::Continue(tool_call)`, possibly transformed, or `PreToolUseDecision::Block { tool_call, reason, user_message }`.

Hooks chain sequentially, each seeing the previous hook's mutated call; the first block wins and aborts the chain, and blocking one call does not affect its siblings in the batch. The model sees a blocked call as a normal tool error reading "blocked by pre_tool_use hook: {reason}". Capability-contributed pre-tool hooks run first, then user-hook specs; the `guardrails` builtin is the config-driven overrider (chapter 07).

Post-tool-exec hooks come from `post_tool_exec_hooks()` and implement `PostToolExecHook::after_exec(tool_call, tool_def, result: &mut ToolResult, context)`, which runs after each execution and before the engine emits the result as an event, to inspect or transform it. Each hook declares a `priority()`: `PostToolExecHookPriority::Guardrail` (0) inspects or blocks output before `Normal` (100) transformation and logging hooks run. Capability hooks run before infrastructure hooks.

One infrastructure hook is always on. `OutputHardLimitHook` caps every tool result at 64 KiB and cannot be removed by capabilities. Cut text ends with "[Output truncated - exceeded 64 KiB limit. Try quiet flags, pipes, or redirect to file.]", truncation respects UTF-8 boundaries, oversized JSON is serialized and truncated to a string, the error field is capped at the same limit, and images over 64 KiB individually or cumulatively are dropped.

Four more hook families shape tool calls around the batch. `ToolDefinitionHook::transform(tools)` rewrites the final deduplicated schema list sent to the model. `ToolCallHook` supplies `narration` for the UI and `transform_for_execution` to strip model-authored fields before execution; the `human_intent` builtin uses it (chapter 07). A `FinalizedToolCallsHook` applies policy over the whole batch, with final schemas, before the assistant message is persisted.

`PostActHook::on_completed(result, tool_definitions)` runs once after the batch and returns declarative `PostActAction` values rather than emitting events. The only action is `EmitToolCallRequested { tool_calls, tool_definitions }`, which emits `tool.call_requested` with synthetic client-side calls.

Output hooks attach to the model output stream. A capability enables them by being added to the agent's capability list; hooks are discovered from configured capabilities, run in capability order, and each receives its own configuration. A streaming guardrail from `output_guardrails()` implements `check(accumulated, delta)` and returns a `GuardrailDecision`. On `Block`, pending text is cleared, the accumulated text is replaced with a canned message, tool calls and reasoning are discarded, and an `output.message.replaced` event is emitted before bad tokens reach the client. The thinking stream is guarded the same way.

An end-of-message guardrail from `post_output_guardrails_with_config()` runs once over the fully assembled reply; when one is present the stream buffers deltas and releases them only if nothing tripped. Model-backed guardrails need a utility LLM service and fail open without one.

`post_output_annotation_hooks_with_config()` annotates the finished reply with citations, `citation_verifier_with_config()` checks those citations against sources, and `filter_response_text(text, config)` post-processes assistant text deterministically before it is persisted, defaulting to identity. The `guardrails` and `prompt_canary_guardrail` builtins are the shipped guardrails and `message_metadata` is the shipped text filter (chapter 07).

## Session records, agent definitions, and configuration layering

### Records

`ExecutionSession` is the portable, execution-facing session value. It carries only what a turn consumes and is separate from the persisted platform record that holds participants, timestamps, and UI metadata; it is the leaf of the harness to agent to session configuration chain. `ExecutionSession::new(id, workspace_id, harness_id)` builds a minimal one with every option empty and `status: SessionExecutionState::Started`; in the default 1:1 case the workspace id mirrors the session id (chapter 08). The fields that override or extend the agent are `model_id` (higher priority than agent and harness), additive `capabilities`, additive client-side `tools`, session-scoped `mcp_servers` (serialized as `mcpServers`, with `mcp_servers` accepted on input; chapter 06), a `system_prompt` override, `max_iterations`, `goal`, `title`, `locale` (BCP 47), `tags`, additive `initial_files`, `network_access`, `parallel_tool_calls`, and `hints`.

A session moves through five states with snake_case wire strings.

- `Started`: created, no turn yet
- `Active`: a turn is running
- `Idle`: waiting for the next input
- `WaitingForToolResults`: the client must submit tool results
- `Paused`: a budget limit was reached (the `budgeting` builtin, chapter 07)

A new peer session is seeded with `SessionSeedMode::Fresh` (empty, lineage recorded), `Fork` (copies conversation events, workspace files, and durable session storage), or `Workspace` (workspace files only); chapters 08 and 09 cover the workspace side. A spawned subagent's `SubagentStatus` is `Spawning`, `Running`, `Completed`, `Failed`, `Cancelled`, `MaxIterationsReached`, or `Sealed`, where sealed is terminal and non-retryable, and is kept distinct from failed (chapter 09). Session-scoped storage is one trait, `SessionStorageStore`, with `set_value`, `get_value`, `delete_value`, and `list_keys` for plain pairs and `set_secret`, `get_secret`, `delete_secret`, and `list_secrets` for values encrypted at rest and decrypted only on read.

`AgentDefinition::new(id, name, system_prompt)` creates the portable agent counterpart with every other field empty. On the wire only `id`, `name`, `system_prompt`, and `capabilities` always appear, `display_name` falls back to `name`, and the MCP servers key follows the same `mcpServers` convention.

### Layering

````rust
let effective = AgentConfigOverlay::fold([
    AgentConfigOverlay::from(&harness),
    AgentConfigOverlay::from(&agent),
    AgentConfigOverlay::from(&session),
]);
````

All three layers share one shape, `AgentConfigOverlay`, with `system_prompt`, `capabilities`, `initial_files`, `network_access`, `default_model_id`, `tools`, `max_iterations`, `parallel_tool_calls`, and `mcp_servers`. `merge(overlay)` applies one layer on top of another with a per-field rule rather than blanket overwrite, and `fold` applies each later layer over the earlier ones.

System prompts are additive, concatenated in order with blank fragments trimmed and dropped, and a session `goal` is appended wrapped in `<session-goal>` tags. Capabilities merge by id: reusing an id replaces that capability's whole configuration in place, unrelated ones keep their order, and new ids are appended. Initial files merge by normalized path (a leading `/workspace/` is stripped and a leading slash ensured), replacing content, encoding, and readonly flag together. Network access can only narrow, because allowed hosts intersect and blocked hosts union, so a session can restrict what a harness allows but never widen it (chapter 06). `default_model_id`, `max_iterations`, and `parallel_tool_calls` follow highest-set-layer-wins, with an explicit zero or false kept. Tools accumulate additively and are deduplicated at build time with the last registration winning. MCP servers merge by logical server name, last wins.

The fully resolved result is a single serializable `RuntimeAgent` with the composed `system_prompt`, `model`, `tools`, `max_iterations`, `temperature`, `max_tokens`, `tool_search`, `prompt_cache`, `driver_options`, `network_access`, `parallel_tool_calls`, and `conversation_context`. When no layer sets an iteration cap, `default_max_iterations()` returns 500, with the session override winning over agent config.

`RuntimeAgentBuilder::from_overlay(layer, registry, ctx)` builds one from the folded overlay in one call. `with_capabilities(ids, registry, ctx)` resolves dependencies in topological order and appends each capability's prompt contribution after the base prompt inside a `<capability id="...">` block, and `with_locale(Some("uk-UA"))` appends a `<locale preference="...">` block. A hosted tool search configuration is cleared automatically for models that do not support it; those models fall back to full tool schemas without a provider error.

## The engine loop: Input, Reason, Act phases, turn planning, and the reason phase

### The loop

Every turn runs through one shared Input, Reason, Act kernel, named `input`, `reason`, and `act` in logs, so the loop shape is the same regardless of provider or tool set. From the `everruns` facade the session drives the whole loop for you, and these mechanics hold whether the host is in-process or durable. The Input phase retrieves the already-persisted user message by id; the `input.message` event was emitted when the message was stored, not by this phase. The Reason phase is a single model call against the assembled context and returns a `ReasonResult` carrying `success`, `text`, `tool_calls`, `usage`, `finish_reason`, `response_id`, and the error with its user-facing classification. The Act phase executes the requested tool calls. The agent builder's `.max_iterations(n)` caps model and tool iterations per turn. Steering messages that arrive mid-turn keep the turn alive with another reason step, as long as the last step succeeded and the cap has not been hit.

Planning is sans-IO. `plan_next_turn(state, outcome, pending_user_message_count, now, facts)` is pure and deterministic: it reads only its arguments and returns a `TurnPlan` plus an ordered `Vec<TurnLifecycleEffect>`, never touching a store, socket, process, event bus, or clock. `ActivityOutcome` is `ProcessInput { turn_id }`, `Reason(Box<ReasonResult>)`, or `Act(ActOutcome)`. The plan is `ScheduleReason(TurnState)`, `ScheduleAct(ActPlan)`, `Complete { stop_reason, error }`, or `WaitForToolResults { resume }`. The effects the host applies in list order are `TurnCompleted`, `SessionIdled`, `TurnFailedWithDisclosure`, `FireTurnEndHooks`, and `WaitingForToolResults`. Progress is kept in a serializable `TurnState` that durable hosts persist between activities and in-memory hosts hold directly; `TurnExecution::new(state)` starts or restores an execution and `into_state()` returns its checkpoint. A turn ends immediately with `EndTurn` and no error when the act phase reports it was blocked because a dependency was archived or deleted.

Transient provider failures before the stream starts are retried with exponential backoff under `with_provider_retry_config(LlmRetryConfig)`, and a stream that fails transiently is retried as long as nothing user-visible (final text or tool calls) has been emitted. Exhausting the time budget yields an error stating the turn is safe to resume rather than an indefinite hang, and transient errors are never surfaced as user-facing error events. A stall with no tokens for the `with_provider_stall_timeout` window (default 120 seconds) is aborted and routed through the same retry path; a stall after partial output is not retried. Retry attempts and total wait are recorded on `llm.generation`. When a reasoning effort is set, extended thinking streams as `reason.thinking.started`, `reason.thinking.delta`, and `reason.thinking.completed`.

### Request controls and context assembly

````rust
let controls = Controls {
    reasoning: Some(ReasoningConfig { effort: Some(ReasoningEffort::High) }),
    ..Default::default()
};
````

Only the most recent user message's controls are read. `reasoning.effort` sets the effort for the next model call; a tool can change it mid-turn through a shared `ReasoningEffortHandle`, and the live handle takes precedence over the message value. Each completed assistant message records the effective effort, which is what a restart resumes from. `speed` selects the service tier as `"flex"`, `"default"`, or `"priority"`, and `verbosity` sets output verbosity. When the target model's profile does not support a requested effort, speed, or verbosity, the engine strips it and logs a warning; no provider error is produced. `model_id` overrides the model for the turn, and when a turn selects a different model than the previous one a `session.model.changed` event is emitted automatically.

Project instructions such as AGENTS.md content (the `agent_instructions` builtin, chapter 07) arrive as a leading user-role `conversation_context` re-resolved every turn without invalidating the cached system prompt, and live facts such as the current time are injected as a trailing user message outside the cached prefix. Multi-actor messages are prefixed with the speaker's display label in brackets. Resuming a session interrupted mid-tool-call inserts a matching tool result for every dangling assistant tool call before the next request, replaying settled results from a durable tool result store when one exists (chapter 08) and otherwise inserting "cancelled - another message came in before it could be completed". Each repair emits `transcript.repaired`, and repair never fails the turn.

## Tool scheduling and the act phase

````rust
let agent = Agent::builder()
    .instructions("Use the tools as needed.")
    .model(Model::simulated("done"))
    .tool(read_config())
    .tool(write_config())
    .parallel_tool_calls(false)
    .build()?;
````

````bash
EVERRUNS_ACT_MAX_TOOL_CONCURRENCY=8 cargo run -p everruns --example canonical_events
````

Every tool call the model requested in one step runs as one batch that always finishes. A tool's failure, timeout, or cancellation is a normal per-call result rather than an abort; the act result reports per-call outcomes plus success and error counts, and an empty batch short-circuits with no events.

Calls execute concurrently by default. Calls that share a non-empty `concurrency_class` in their `ToolHints` run one at a time in arrival order, while calls in different classes or with no class run in parallel; read-only tools declare no class and always parallelize. A tool sets its class with `with_concurrency_class("session_workspace")`. `.parallel_tool_calls(false)` on the agent builder, also available per harness and per session, forces the batch strictly sequential in arrival order and ignores classes; unset or `true` keeps the class-aware schedule. The number of calls executing at once within one act phase is capped at 32 by default and overridden per process with `EVERRUNS_ACT_MAX_TOOL_CONCURRENCY`; the value is clamped to at least 1, and non-numeric values silently fall back to the default. Results return in the order the model emitted the calls regardless of which finished first.

A tool marked `cpu_bound: true` is offloaded onto its own task, which keeps long synchronous work from starving I/O-bound tools in the same batch. Cancelling a turn drops every in-flight tool future and aborts every offloaded task; nothing leaks past cancellation. Each tool receives a per-call cancellation token through `ToolContext.cancellation` that fires when the call ends or the turn is cancelled, so detached work such as a child process is signalled.

A tool declares its replay-safety class as `SideEffectClass::Pure`, `Idempotent`, or `AtMostOnce` (the default); after a worker failure, pure and idempotent tools re-run safely while an at-most-once tool returns an "interrupted" error, since re-running it would risk a double side effect. With a durable tool result store attached (chapter 08), each call is claimed before dispatch and settled after, and already-settled results replay without re-running the tool or re-emitting `tool.started`.

When the model calls a tool that is not defined, it sees `tool.completed` with status `error` and the text "Tool definition not found: {name}". A tool can return images, which are delivered to the model as native image parts appended after the text result. Tools emit `tool.progress` mid-execution because every tool context receives the event emitter and its own call id. The merged network access list restricts which URLs tools may reach, and a per-organization outbound rate limit can deny a call with a tool error and no events. Narration for the UI comes from hook narration first, then the tool's own `narrate`, then a generic display-name fallback.

Client-side tools are not executed by the engine: they are handed back through `tool.call_requested` and the worker pauses until results arrive. When a tool reports that a third-party connection is required, a built-in hook emits one synthetic `setup_connection` call per distinct provider (chapter 06). Custom execution backends implement `ToolExecutor::execute(tool_call, tool_def)`, with optional `execute_with_context`, `execute_batch`, and `execute_parallel` helpers.

## Compaction mechanics

````rust
while let Some(event) = events.recv().await? {
    if let SessionEventKind::Other { event_type } = &event.kind {
        if event_type == "context.compacted" {
            let data = &event.canonical_json()["data"];
            println!(
                "compacted via {}: {} -> {} messages",
                data["strategy_used"], data["messages_before"], data["messages_after"]
            );
        }
    }
    if event.kind.is_terminal() {
        break;
    }
}
````

A conversation that outgrows the model's context window is compacted rather than failed. The policy that governs compaction is supplied by a capability through `compaction_policy()`. The policy's `strategy` determines which stages the engine may use: `Auto` (every stage), `Native`, `ObservationMasking`, or `Summarization`. A custom `CompactionPolicy` controls token estimation, pressure thresholds, masking, trimming, and summary composition through methods such as `estimate_total_tokens`, `should_compact_proactively`, `apply_observation_masking`, and `aggressive_trim`.

Compaction runs proactively before a request exceeds the window, using the policy's window-pressure check against the resolved context window, and can also run early for cost pressure based on estimated tokens, accumulated raw tool-result bytes, and prior usage. The window resolves from the driver, then the model profile, then a default of 128,000 tokens. When a provider rejects a request as too large, the engine runs a compaction cascade and retries the model call; the turn does not fail. Without a policy it logs that compaction is not enabled and returns the error.

The cascade tries the provider's native compaction endpoint when the driver supports it, passing the agent's system prompt as instructions. Next comes observation masking, which hides old tool results from the model and needs no checkpoint store, then summarization of the older conversation with a model call at temperature 0 and 2000 max tokens that keeps the 10 most recent messages verbatim, and finally aggressive trimming as the last resort when no other strategy ran or the message count did not at least halve. Every strategy preserves the first system message. A compaction counts as applied only when it materially reduces the request, by at least 5 percent of the before size or 32 units, whichever is larger; otherwise a skipped event with reason no material reduction is emitted and the original provider error stays authoritative. A repeated proactive native attempt on the same input requires a new source sequence and estimated token growth of at least the larger of 4,096 tokens or one twentieth of the previous estimate, and after restoring from a checkpoint proactive compaction waits for 4 new messages before re-arming. Sessions using native reasoning state are never degraded by summary or trim; the skip reason is guard rejected and the context error surfaces.

Four events cover the lifecycle, and every pressured evaluation closes with one, and only one, of the last three. `context.compacting` carries the `reason` (`proactive_budget`, `request_too_large`, or `manual`), the `trigger` (`ContextBudget`, or `CostPressure` when there was no window pressure), the `strategy`, the model and provider, and the messages, tokens, and bytes before. `context.compacted` carries `checkpoint_id`, `strategy_used` as the exact chain such as `native`, `masking+trim`, or `observation_masking+summarization+aggressive_trim`, the before and after counts, `duration_ms`, and one `steps` entry per strategy. `context.compaction.skipped` carries a `CompactionSkipReason` of `StrategyExcludesNative`, `DriverUnsupported`, `CheckpointStoreUnavailable`, `CooldownActive`, `NativeReturnedNone`, `NoMaterialReduction`, or `GuardRejected`. `context.compaction.failed` carries a `CompactionFailStage` of `NativeCompaction`, `CheckpointInstall`, or `Summarization` plus the error. Compaction cost and token deltas are attached to the generation's `llm.generation` metadata as `LlmCompactionInfo`. With a durable checkpoint store (chapter 08), resuming from a checkpoint reloads only messages after it and injects the summary as a system message wrapped in `[CONVERSATION_SUMMARY]` markers; compacted context is applied only after the durable install succeeds.

## Error policy and disclosure

````rust
let controls = Controls {
    error_disclosure: Some("generic".to_string()),
    ..Default::default()
};
````

````rust
if let SessionEventKind::OutputCompleted { .. } = &event.kind {
    let data = &event.canonical_json()["data"];
    if let Some(code) = data["error_code"].as_str() {
        eprintln!("model call failed: {code} (disclosure {})", data["error_disclosure"]);
    }
}
````

How much detail about a model or provider failure reaches the end user is governed by an `ErrorDisclosure` mode. `Generic` collapses every blocking error into one generic, localizable processing-error message with no fields, for public-facing agents. `Standard`, the default, returns a stable error code plus structured interpolation fields. `Detailed` adds a `detail` field carrying the underlying driver error text trimmed to 1000 characters, for trusted surfaces such as coding-agent harnesses. The ceiling is set by the `error_disclosure` builtin on the agent (chapter 07); with no capability configured the ceiling is `Standard`. A client narrows disclosure per message by sending `error_disclosure` on the user message's `Controls` as `"generic"`, `"standard"`, or `"detailed"`, and can narrow but never widen past the operator-selected ceiling. The same `Controls` carry `locale` (BCP 47) to override backend-authored strings and prompts for one turn, and `hints`, which shallow-merge over session hints with per-message values winning.

A failed model call does not crash the turn. It surfaces as a normal assistant message carrying the disclosed, classified error, and the `output.message.completed` event records `error_code`, `error_fields`, and `error_disclosure`, while the message metadata records `error_disclosure` and `source_error_code`. Each class of failure gets a distinct, actionable message: a generic processing error, a provider outage, rate limiting, context too long (asking the user to start a new session or reduce context), provider misconfiguration, no model configured, and provider out of credits or quota.

The engine recognizes its own placeholder replies through the stable `error_code` in message metadata, using the constants in `user_facing_error_codes` such as `REQUEST_TOO_LARGE`, `PROVIDER_RATE_LIMITED`, `MODEL_NOT_CONFIGURED`, and `PROCESSING_ERROR`, alongside the budget, quota, usage-limit, misconfiguration, availability, and dependency codes. Placeholders are treated as failed turns rather than genuine output and are stripped from model input on later turns; a message with tool calls is never a placeholder. Capabilities can react to terminal model errors through `llm_error_hook()` and enrich the user-facing error fields, which is how the `usage_limit_auto_continue` builtin schedules an auto-continue after a usage limit resets (chapters 07 and 09).

## Observability: reason-phase spans and request options

````rust
if let SessionEventKind::ModelGeneration { model, input_tokens, output_tokens, cost_usd, duration_ms, success, .. } = &event.kind {
    println!("{model}: {input_tokens:?} in, {output_tokens:?} out, {cost_usd:?} USD, {duration_ms:?} ms, ok={success}");
    let options = &event.canonical_json()["data"]["metadata"]["request_options"];
    println!("effort={} stream={}", options["reasoning_effort"], options["stream"]);
}
````

Reason phases are traced hierarchically with OTel-style trace and span ids. The turn is the parent span and its id is the `trace_id`, each reason phase gets its own span, and child events such as `llm.generation` and the output message events hang off the reason span; each act phase and each tool call likewise get spans under the turn. The `EventContext` on every canonical envelope carries `turn_id`, `input_message_id`, `exec_id`, `trace_id`, `span_id`, and `parent_span_id`, in a Braintrust-compatible format. Provider API requests carry request metadata correlating them with `session_id`, `harness_id`, `turn_id`, `exec_id`, and `org_id`, plus `agent_id` and `model_id` when present. Each assistant message is stamped with `model`, `reasoning_effort`, `provider`, and `response_id` metadata so a chat UI can deep-link to the provider's trace for that message.

A `capability.usage` snapshot event lists, by id and name only, which capabilities were `Resolved` and which tools were `Exposed` for the reason phase. The other `CapabilityUsageKind` values are `Configured`, `Invoked`, and `EffectRan`, and prompts, messages, tool arguments, and results are never allowed in these reporting facts.

The typed `ModelGeneration` kind is enough to track spend and latency without touching the canonical surface. The canonical `llm.generation` payload adds the full picture: `messages` including the system prompt, `tools`, `output`, and an `LlmGenerationMetadata` with `model`, `provider`, `usage`, `duration_ms`, `time_to_first_token_ms`, `success`, `error`, `finish_reasons`, `response_id`, `retry` as `LlmRetryInfo { attempts, total_wait_ms }`, `compaction`, and `request_options`. `LlmRequestOptions` records the exact sampling parameters sent with the call: `temperature`, `max_tokens`, the effective `reasoning_effort`, `stream` (always `true`, since the reason phase always drives the provider through its streaming endpoint), `tool_search` with `enabled` and `threshold`, `provider_options` such as OpenAI previous-response chaining, and `metadata` copied verbatim from the call configuration. Its `prompt_cache` field, an `LlmPromptCacheInfo { enabled, strategy, provider_mode }`, appears only when caching was enabled and names the mechanism in effect: `implicit`, `explicit`, or `prompt_cache_key` for OpenAI, `cache_control` for Anthropic, and `cached_content` or `implicit` for Gemini. The tool search builtins and the `prompt_caching` builtin that set these are configured in chapter 07. `TokenUsage` on every event reports disjoint prompt buckets, `input_tokens`, `cache_read_tokens`, and `cache_creation_tokens`, plus `output_tokens`, `actual_cost_usd`, `estimated_cost_usd`, and an `effective_cost_usd` that prefers actual over estimate; consumers must not re-derive non-cached input by subtracting cache reads.

*2026-09-17 03:24 - claude-fable-5.1*
