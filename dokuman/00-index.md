<!-- source: everruns/everruns @ 6cf2c15e5 (crate everruns 0.20.0). The working tree fast-forwarded to 7d9e07a0b (0.21.1) during extraction; identifiers and examples were audited against 6cf2c15e5. -->
# The Everruns Framework

Everruns is a Rust crate for building AI agents inside the program you already have. You describe an agent as a value, hand it to an engine, and run conversations from ordinary Rust code. The default build compiles offline against a simulated model, so your first agent runs before you have an API key. Anything beyond that, a shell, a network path, a script interpreter, an external tool server, is a boundary you cross by name with a feature flag and a builder call. The ten chapters below document the crate from its four core nouns down to the contracts a downstream crate implements.

## How to read this set

Thirty seconds gets you chapter 01: what an agent, an engine, a session, and a turn are, why the build is offline by default, and where the framework ends and the hosted platform begins. Five minutes gets you chapter 02 as well, at the end of which you have added the crate, built an agent with a typed tool, and printed a response, first offline and then against OpenAI. The remaining eight are the full exposition. Read them in order; each introduces its concepts before using them and assumes only the chapters before it. Persistence (08) precedes workspaces (09) because git workspaces, schedules, and background tasks all rest on the `local` feature.

## The chapters

[01 What Everruns Is](01-what-is-everruns.md). The identity chapter. Agent, Engine, Session, and Turn as types with a documented data flow, then capability, harness, host, and workspace. The design choices that shape everything after it are stated here once: tools are plain async functions under one attribute, models are provider-neutral, capabilities are explicit opt-ins, and every turn emits events you can observe and cancel.

[02 Your First Agent](02-your-first-agent.md). From `cargo add everruns` to a printed reply. Default and opt-in feature flags with what each pulls in, `Agent::builder()`, `Engine::new()`, `send_and_wait`, `turn.response`, the shipped examples and their required features, and the `--no-default-features` build for a strict dependency policy.

[03 Tools and the Macro](03-tools-and-the-macro.md). `#[everruns::tool]` in full: attribute options, accepted signatures, and every compile-time rejection as a failing snippet with its exact error. Beneath the macro sit `FunctionTool`, the core `Tool` trait and registry, and the capability contract (ids, references, specs, `IntoCapability`) that packages several tools with instructions into one reusable unit. Code-defined capabilities through `Definition` and `Handler` end the chapter.

[04 Models and Providers](04-models-and-providers.md). Offline simulator first, then OpenAI through the `openai` feature and both of its protocols, then Anthropic, Bedrock, Gemini, Fireworks, MAI, Meta, and OpenRouter with each vendor's env vars and constructor. Readers who want to understand or extend the provider layer get the neutral `everruns-provider` contract: provider values, the driver registry, credentials, model specs, request controls, error classification, URL safety, and a custom `ChatDriver`.

[05 Sessions, Events, and Hooks](05-sessions-events-and-hooks.md). Where a built agent runs. Creating and resuming sessions, sending and steering, the turn record and its stop reasons, history pages, the canonical event stream with a live subscription example, cooperative cancellation, lifecycle hooks with one example per hook point, tool and output hooks. The engine loop follows: Input, Reason, Act, tool scheduling and its concurrency cap, compaction mechanics, error policy, observability spans.

[06 Capability Boundaries](06-capability-boundaries.md). Each opt-in boundary from the session filesystem outward. The Bashkit sandboxed shell with its CLI builtins and hook dispatch; web fetch; DuckDuckGo and Brave search; the Lua sandbox and code mode; MCP across four sections for attaching servers, transports and protocol, OAuth and credentials, and URL elicitation. Every outbound request from any of these passes one host egress boundary, and the last section is about that.

[07 Built-in Capabilities](07-built-in-capabilities.md). The `everruns-builtins` catalog grouped by what each builtin does for the agent: instructions and session context, cost and progress limits, safety guardrails and tool approval, tool discovery and tool-call repair, context management including compaction and infinity context, skills and the SKILL.md format, task lists and subagent delegation, user hooks and hook bundles. Every entry gives the capability id, its config keys and defaults, and an example where the syntax is documented.

[08 Persistence and the Local Runtime](08-persistence-and-local-runtime.md). The engine-lifetime in-memory default against the `local` feature. `LocalConfig`, the SQLite session catalog, the crash-durable event log with a resume-after-restart example, `LocalProfile` and `LocalBackends` and the runtime builder, error variants, the `a2a` opt-in, execution loading contracts, and the host backends a single-node deployment satisfies.

[09 Workspaces and Background Work](09-workspaces-and-background-work.md). Workspaces, environments, heads, and the default `/workspace`; compute targets and containment; `WorkspacePolicy` with its presets and deny-by-default scopes; roots and path resolution; the session filesystem contract; git workspaces; the security rules. Time enters through session-owned background work and tasks, the task registry, cron schedules and the schedule runner, and wake routing for host-driven sessions.

[10 Testing, Plugins, and Extension](10-testing-plugins-and-extension.md). A complete offline test opens it, then the llmsim scripting surface for replies, tool calls, delays, and failures. Plugin manifests and the compiler, the three kinds of pack, the event log SPI, the external-consumer fixture as the template for a downstream crate, `everruns-host` builders for hand-assembled hosts, architecture and crate layering, the examples catalog, and a complete feature flag reference.

## Provenance

Capability statements were extracted from 400 files: the `everruns` crate, the crates its features pull in, the framework documentation, the design notes, and the root examples. They were then tiered, verified against the repository, and written under a single evidence discipline. Every identifier, config key, feature flag, env var, and default in these pages was checked against source at the commit named in each file's header. Where documentation and source disagreed, source won and the page says so.

*2026-09-17 03:27 - claude-fable-5.1*
