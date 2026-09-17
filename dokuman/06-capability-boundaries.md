<!-- source: everruns/everruns @ 6cf2c15e5 (crate everruns 0.20.0). The working tree fast-forwarded to 7d9e07a0b (0.21.1) during extraction; identifiers and examples were audited against 6cf2c15e5. -->
# Capability Boundaries

An agent built from the default `everruns` crate can read and write files inside its own session, and that is the whole of its reach. A shell, a network path, a script interpreter, or an external tool server is a boundary you cross on purpose, once in `Cargo.toml` with a feature flag and once per agent with a marker or builder value, and each boundary arrives with its tool names, config keys, hard limits, and enforcement points already decided. The sections below walk those boundaries from the session filesystem outward and end at the one place every outbound request from any of them must pass: the host egress boundary.

## The opt-in boundary model and feature flags

````toml
[dependencies]
everruns = { version = "0.20.0", features = ["bashkit", "web-fetch"] }
````

````rust
use everruns::prelude::*;

let agent = Agent::builder()
    .instructions("Inspect the repository and summarize the build.")
    .model(Model::simulated("done"))
    .capability(FileSystem)
    .capability(BashkitShell::new())
    .capability(WebFetch::new())
    .build()?;
````

The `features` array on the `everruns` dependency compiles integrations in. The `.capability(...)` calls activate them on one agent. Enabling `bashkit` and `web-fetch` in `Cargo.toml` makes the shell and web fetch available to the process, and an agent that lists neither still has only its filesystem. Each integration is a separate crate pulled in by its feature (`everruns-integrations-bashkit`, `everruns-integrations-web-fetch`, and so on), and each is attached through a small type re-exported at the crate root and in the prelude that converts into a capability reference (chapter 03) holding the capability id and any JSON config. `FileSystem` and `DuckDuckGo` are zero-config markers; the rest are builders or constructors with a few options each.

The default feature set is `macros`, `capabilities`, `builtins`, and `filesystem`, so a fresh project already has the session filesystem registered and can attach `FileSystem` with no change to `Cargo.toml`. Everything else is off until named.

- `filesystem` (default on): a host-provided, session-scoped filesystem. Capability id `session_file_system`; attach with `FileSystem`.
- `bashkit`: a sandboxed shell. Capability id `bashkit_shell`; attach with `BashkitShell::new()`. HTTP from scripts stays gated by capability config and egress policy. This feature also turns on `filesystem` at the host level, because the shell runs against the session filesystem.
- `web-fetch`: FetchKit requests through the host egress contract. Capability id `web_fetch`; attach with `WebFetch::new()`.
- `duckduckgo`: DuckDuckGo instant answers. Capability id `duckduckgo`; attach with `DuckDuckGo`.
- `lua`: a vendored Lua 5.4 sandbox. Capability ids `lua` and `lua_code_mode`. Also requires `FEATURE_LUA=true` in the environment at runtime; the compile-time feature alone registers nothing.
- `mcp`: remote HTTP MCP servers through the host egress contract. Attach servers with `McpServer::http(...)` on the builder's `mcp_server` method.
- `mcp-stdio`: local-process MCP servers over stdio; implies `mcp`. A stdio server's command, arguments, and environment are trusted host configuration, and the hosted server and worker builds do not enable it.

A host that assembles its own capability registry calls `runtime_capability_registry()`, which returns a registry reflecting exactly the Cargo features enabled on the host crate, optionally including the builtins bundle (a set of host-provided utility tools, chapter 07). If you are migrating code that imported these types from the core crate, the filesystem capability and tools are now in `everruns_integrations_filesystem`, the shell in `everruns_integrations_bashkit`, web fetch and its bot-auth helpers in `everruns_integrations_web_fetch`, Lua and code mode in `everruns_integrations_lua`, and the MCP capability and id helpers in `everruns_mcp`. Application code that goes through the `everruns` prelude is unaffected.

## Session filesystem tools

````rust
use everruns::prelude::*;

let agent = Agent::builder()
    .instructions("Keep running notes in /workspace/notes.md and answer from them.")
    .model(Model::simulated("noted"))
    .capability(FileSystem)
    .build()?;
````

Every session gets an isolated filesystem rooted at `/workspace`, files persist for the session's duration, and no session can see another's files. The zero-config `FileSystem` marker is how one agent opts into it; the capability id is `session_file_system`, it has no dependencies, and because the default `filesystem` feature already registers it, this builds with no change to `Cargo.toml`. Its tools only ever operate through the filesystem the host supplies, so the host's path and mount policy holds for every call.

`read_file`, `read_many_files`, `write_file`, `edit_file`, `list_directory`, `grep_files`, `delete_file`, and `stat_file` are the eight tools the model sees. `/workspace/notes.md`, `/notes.md`, and a plain relative `notes.md` all name the same file. By default the system prompt tells the model the workspace root is `/workspace`; when a host store opts into backend-native display (real host paths such as `/host/repo`, covered with workspaces in chapter 09), the prompt guidance and every tool schema are updated to match. The prompt contribution stays under 1000 bytes and includes hints on keeping reads small.

### Reading

`read_file(path, offset, limit)` returns numbered lines plus `total_lines`, `lines_shown`, `truncated`, `size_bytes`, and a `content_hash` of the form `sha256:...`. `offset` defaults to 0 and is 0-indexed; `limit` defaults to 2000 lines with a minimum of 1. When no explicit limit is given the defaults follow content type: log files (`.log`, `.out`) return the last 500 lines, CSV and TSV return 100 lines with the header row prepended when the read starts past line 0, minified files return about 20 lines, and known binary extensions return metadata only. An explicit `limit` always wins.

Reading a PNG, JPEG, GIF, or WebP file returns the image natively to the model with a media type sniffed from the bytes. Other binary files never return inline base64. A read cut by the line cap includes `next_offset` and a hint of the form "call read_file with offset=N to resume from line N+1"; a read cut mid-line by the byte cap returns a size-cap envelope with no resume.

`read_many_files(paths, offset, limit)` reads 2 to 10 independent files in one call and returns `{"results": [...], "count": N}` with per-file results or errors in request order. Aggregate output is capped at 60 KiB; past the cap, remaining entries become "read this file individually" errors rather than a truncated blob.

### Writing, editing, and searching

`write_file(path, content, encoding)` creates or overwrites a file and creates parent directories automatically. `encoding` is `text` (default) or `base64`. The response includes `path`, `size_bytes`, `created`, and `content_hash`.

````json
{
  "path": "/workspace/app.py",
  "expected_hash": "sha256:1c4d...",
  "edits": [
    { "old_text": "return 'Hello, World!'", "new_text": "return 'Hello from Everruns!'" }
  ]
}
````

`edit_file(path, expected_hash, edits[])` applies exact text replacements to an existing text file. `expected_hash` is the `content_hash` from the last `read_file` or `write_file`, and the tool compares and swaps against the current content. If the file changed underneath the model and the hunks still match exactly and uniquely, they are rebased and the response reports `rebased: true`; a conflicting change fails with an error that tells the model to read the file again. A missing or ambiguous match leaves the file untouched and returns an explanatory error, and so do overlapping hunks; fuzzy matching is deliberately absent. The tool preserves a UTF-8 BOM and the file's original line ending style, rejects binary files and directories, and every success returns a unified diff (capped at 16000 characters, marked `diff_truncated: true` when cut), `first_changed_line`, `applied_edits`, and the new `content_hash`. A legacy top-level `old_text`/`new_text` pair is still folded into `edits[]`; new callers should always send `edits[]`.

`list_directory(path, offset, limit)` returns `name`, `path`, `is_directory`, `size_bytes`, and `is_readonly` per entry plus paging fields; `path` defaults to the workspace root, `limit` defaults to 200 and maxes at 1000. `grep_files(pattern, path_pattern, before_context, after_context, offset, limit)` searches contents with a Rust regex and returns `path`, `line_number`, and `line` per match; `limit` defaults to 200 and maxes at 1000, context lines default to 0 and max at 20, and output is capped at 64 KiB with resume metadata. `path_pattern` takes globs such as `*.txt` or `src/**/*.rs`, and non-zero context returns merged numbered blocks instead of flat matches. `delete_file(path, recursive)` defaults `recursive` to false and returns a correctable error for a non-empty directory without it. `stat_file(path)` returns `exists`, `is_directory`, `is_readonly`, `size_bytes`, `created_at`, and `updated_at` in RFC 3339, and a missing path yields `exists: false` rather than an error.

## Sandboxed Bashkit shell: the bash tool, sandbox limits, and shell egress

````toml
[dependencies]
everruns = { version = "0.20.0", features = ["bashkit"] }
````

````rust
use everruns::prelude::*;

let agent = Agent::builder()
    .instructions("Use the shell to inspect /workspace and report what you find.")
    .model(Model::simulated("inspected"))
    .capability(FileSystem)
    .capability(BashkitShell::new())
    .build()?;
````

Commands run inside Bashkit, a virtual Bash interpreter written in Rust and embedded in-process, built for untrusted scripts in multi-tenant agent environments. There is no `/bin/bash` process, no direct network stack, no subprocess spawning, and no filesystem beyond the session workspace. `BashkitShell::new()` adds the `bashkit_shell` capability (legacy alias `virtual_bash`) without hand-writing an id and JSON config. It depends on `session_file_system`, so `FileSystem` goes alongside it, and shell commands read and write the same files as the eight filesystem tools.

### The bash tool and the sandbox

The tool is named `bash` and its one required parameter, `commands`, is the script: one command or a multi-line program. `working_dir` (default `/workspace`) sets the starting directory per call, and a path learned from `read_file` can be passed straight to `cd`. `timeout_ms` (default 30000, capped at 60000) bounds wall-clock time; on timeout the command is cancelled at the next command boundary and any partial output captured so far is returned alongside the error. `output` (default `auto`) sets verbosity: `auto` returns a compact summary on success and a normal-sized diagnostic window on failure. Every call returns `stdout`, `stderr`, `exit_code`, `success`, `truncated`, and `total_lines`, while the raw output is persisted separately (chapter 07). Output also streams live through `tool.output.delta` events, and long scripts can run detached in the background (chapter 09).

Inside the sandbox the session filesystem is mounted at root, with `/workspace` as the default working directory, and redirects create parent directories automatically. Paths such as `/etc` or `/tmp` do not exist and cannot be written. `chmod` and `touch` succeed as no-ops because the session filesystem tracks no Unix permissions and files are executable by default, and symlinks are unsupported. `grep -r` uses the store's indexed search rather than a per-file scan. About 85 built-in commands are available, from `echo` and `cat` through `sed`, `awk`, `jq`, and `sort` to `tar` and `gzip`, plus the full POSIX shell language, and built-ins support `<command> --help`, which the tool description tells the model. The `bashkit-repo-agent` example maps a real host directory to `/workspace` through a declared workspace and workspace policy (chapter 09), so the shell operates over that directory.

Hard limits bound every run at 1,000 commands, 10,000 loop iterations, function depth 100, 1 MB script input, AST depth 100, a 5 s parser timeout, 10 MB memory, and the per-call wall-clock timeout. Every call sees the same environment, with user and host `everruns`, `HOME=/home/agent`, `SHELL=/bin/bash`, `PATH=/usr/local/bin:/usr/bin:/bin`, `WORKSPACE=/workspace`, and `LANG` following the session locale (default `en-US`). Shell activity is auditable through structured `tracing` events on the `bashkit.hook` target, tagged with the session id. Argument values and command output are never logged; a completion line records duration, exit code, commands executed, file reads and writes, and byte counts.

### Shell HTTP and egress

````rust
let agent = Agent::builder()
    .instructions("Fetch release notes with curl and summarize them.")
    .model(Model::simulated("summarized"))
    .capability(FileSystem)
    .capability(BashkitShell::new().enable_http(true))
    .build()?;
````

Outbound HTTP from `curl` and `wget` is off by default; without the flag the interpreter has no network path at all. `enable_http(true)` on the builder, or the config key `enable_http: true` on a hand-written capability reference, turns it on. The key is a boolean with default `false`, and a non-boolean value is rejected at config validation. With HTTP enabled, every request and every redirect hop is routed through the host egress boundary, and there is no fallback that bypasses it: if HTTP is enabled but the host supplies no egress service, the shell stays offline and a warning is logged. Bashkit keeps its own pipeline on top, including a DNS and private-IP SSRF precheck with pinned addresses, per-hop redirect validation, bot-auth signing, and response caps.

Scripts see policy denials as curl's native `access denied` failure (exit 7). Timeouts exit 28 and other transport errors exit 1. A response body over 10 MB exits 63 ("response too large"). When `BOT_AUTH_SIGNING_KEY_SEED` is set (optionally with `BOT_AUTH_AGENT_FQDN` and `BOT_AUTH_VALIDITY_SECS`), outbound shell requests are transparently bot-auth signed under the same contract as web fetch. The Bashkit ecosystem page states that the shell has no network access; that statement is stale, and the `enable_http` opt-in described here is the current behavior. The integration pins Bashkit 0.18.0 and uses the `aws-lc-rs` crypto backend rather than `ring`.

## Shell CLI builtins and bash hook dispatch

### An application command tree inside the shell

````rust
use async_trait::async_trait;
use everruns_integrations_bashkit::cli::{
    CliCommandSource, CliCommandSourceHandle, CliCommandSpec, CliRoute,
};
use serde_json::{json, Value};

const LIST: CliRoute = CliRoute::new(&["fleet"], "list").with_examples(&["everruns fleet list"]);
const SCALE: CliRoute = CliRoute::new(&["fleet"], "scale")
    .with_examples(&["everruns fleet scale --name api --replicas 4"]);

#[async_trait]
impl CliCommandSource for FleetCommands {
    fn specs(&self) -> Vec<CliCommandSpec> {
        vec![
            CliCommandSpec {
                wire_name: "list_services".into(),
                description: "List services and their replica counts.".into(),
                route: LIST,
            },
            CliCommandSpec {
                wire_name: "scale_service".into(),
                description: "Set a service's replica count.".into(),
                route: SCALE,
            },
        ]
    }

    fn node_about(&self) -> Vec<(String, String)> {
        vec![("fleet".into(), "Services this deployment runs, and their scale.".into())]
    }

    async fn dispatch(&self, wire_name: &str, params: Value) -> Result<String, String> {
        match wire_name {
            "list_services" => Ok(json!({ "services": self.rows() }).to_string()),
            "scale_service" => {
                let name = params.get("name").and_then(Value::as_str).ok_or("missing --name")?;
                let replicas = self.scale(name, &params)?;
                Ok(json!({ "name": name, "replicas": replicas, "scaled": true }).to_string())
            }
            other => Err(format!("unknown command {other}")),
        }
    }
}
````

With this source installed, the model can type `everruns fleet scale --name api --replicas 4` inside the `bash` tool and get back `{"name":"api","replicas":4,"scaled":true}`. A Framework application exposes its own operations to the agent as an `everruns <noun> <verb> [--flags]` command tree, so the model sees a discoverable CLI rather than a flat namespace of wire names. The application implements one trait, `CliCommandSource`. `specs()` returns the catalog and `dispatch()` receives a wire name with parsed flags. `root()` is optional and rebrands the root token (`acme invoices send`); `node_about()` and `usage()`, also optional, supply one-line noun summaries and usage text.

Each `CliCommandSpec` pairs a wire name and description with a static `CliRoute`: a noun path and a verb, plus optional example invocations shown in help. A command joins the tree only by declaring a route, so internal operations cannot leak into the agent-facing surface. The source reaches a session as a `CliCommandSourceHandle` inserted on the tool context extensions through the runtime builder's `with_tool_context_extensions_factory` (chapter 08). A host that supplies none gets no builtin and no prompt claim about it.

Both `--flag value` and `--flag=value` reach `dispatch` as members of one JSON object. A bare `--flag` is a boolean `true`, values starting with `[` or `{` that parse as JSON arrive structured, `true` and `false` arrive as booleans, and everything else stays a string so the source's own schema does type validation.

`everruns --help` lists the top-level nouns with summaries, `everruns fleet --help` lists that noun's children and verbs, and `everruns fleet scale --help` renders the description, usage, examples, and wire name, so help output is bounded by the tree's shape rather than by a cap. `--help` or `-h` anywhere in an invocation takes precedence over execution. An invocation that does not resolve becomes a help call that prints the live children under the deepest known scope and exits non-zero, naming the first unresolvable segment. In a scripted-tool host, `everruns agents list --limit 10` is rewritten to `list_agents --limit 10` at statement boundaries.

### Bash hooks run in the sandbox

````bash
printf '%s\n' "$EVERRUNS_HOOK_TOOL_NAME:$EVERRUNS_HOOK_TOOL_CALL_ID" >> /workspace/.audit.log; echo '{}'
````

That is a complete `post_tool_use` hook command from the `audit-every-tool` bundle: it appends one line per completed tool call and ends with `{}`, which allows. User-defined hooks (chapter 07) whose command is a bash script run inside the same sandboxed Bashkit interpreter as the `bashkit_shell` capability and never spawn a host process. `BashkitShellHookDispatcher::new(store)` builds the dispatcher for a session from the session file store and `BashHookExecutor::with_dispatcher(command, env, dispatcher)` plugs it in; one dispatcher can serve several sessions if the store is shared. Without the `bashkit` feature, bash hooks fail with "bash hooks require the everruns-host `bashkit` feature".

A bash hook attaches to `pre_tool_use` (gate a call before it runs), `post_tool_use`, `user_prompt_submit` (payload `data` has `message`), `turn_end` (advisory; `data` has `success`), or `session_start` (`data` has `agent_id`). A `pre_tool_use` hook can be scoped with a matcher of `tool_name`, `args_jsonpath`, and `deny_regex`, so a script that blocks `rm -rf` in `bash` commands never runs for `ls -la`. Hooks see cwd `/workspace` and hostname `everruns-hook`, and they read and write the agent's own `/workspace` files.

The full payload arrives as JSON in `$EVERRUNS_HOOK_PAYLOAD_JSON` (always set) and as a file at `$EVERRUNS_HOOK_PAYLOAD_PATH`, cleaned up after the run; Bashkit exposes no process stdin to user scripts. Payload fields are `event` (snake_case, such as `post_tool_use`), `hook_id`, `session_id`, `turn_id`, `org_id`, `agent_id`, `ts`, and event-specific `data`; tool events include `tool_name`, `tool_call_id`, and `arguments`. Convenience scalars such as `$EVERRUNS_HOOK_EVENT` and `$EVERRUNS_HOOK_TOOL_NAME` avoid JSON parsing, and operator-supplied `env` from the hook spec may intentionally override them.

````bash
printf '%s' '{"decision":"block","reason":"nope","user_message":"blocked"}'
````

A hook decides by printing JSON to stdout. `{"decision":"block","reason":"...","user_message":"..."}` blocks. `{"decision":"mutate","patch":{...}}` rewrites tool arguments for `pre_tool_use` or `message` for `user_prompt_submit`. `{}` or empty stdout with exit 0 allows. A non-zero exit with no JSON blocks with stderr as the reason, and non-JSON stdout is an error outcome, never a silent allow. `timeout_ms` on the spec caps runtime (the bundles use 5000); a timed-out script is reported as an error, and the spec's `on_error` (warn or block) determines what that means for the call. Hook scripts run under tighter limits than the agent shell, at 200 commands, 2,000 loop iterations, function depth 32, 64 KiB script input, AST depth 64, a 2 s parser timeout, and 10 MB memory. Output over the spec's `max_output_bytes` is an error rather than a truncated decision.

## Web fetch: the web_fetch tool, content handling, and fetch egress policy

````toml
[dependencies]
everruns = { version = "0.20.0", features = ["web-fetch"] }
````

````rust
use everruns::prelude::*;

// WebFetch needs no credential of its own; request signing, when wanted, is
// configured from BOT_AUTH_SIGNING_KEY_SEED in the environment (see below).
let agent = Agent::builder()
    .instructions("Read the page the user names and answer from its text.")
    .model(Model::simulated("read"))
    .capability(WebFetch::new())
    .build()?;
````

`WebFetch::new()` is the whole attachment for the `web_fetch` capability, and the `hackernews-reader` example shows how far that goes: it builds a HackerNews browsing agent with zero custom tool code by letting `web_fetch` call the HN API directly. The capability is FetchKit-backed, has no dependencies, and exposes one tool. File download is off in that default: the only config key is the boolean `enable_file_download`, default `false`, and `WebFetch::new().enable_file_download(true)` sets it so fetched content can be saved into the session workspace. A non-boolean value is rejected at config validation.

Advanced hosts construct the capability directly with `WebFetchCapability::new(bot_auth)`, passing an optional server-wide `BotAuthConfig`, or with `WebFetchCapability::from_env()`, which reads `BOT_AUTH_SIGNING_KEY_SEED` (required to enable signing; a base64url 32-byte seed), `BOT_AUTH_AGENT_FQDN` (optional), and `BOT_AUTH_VALIDITY_SECS` (optional, default 300). When configured, every outbound request is signed with Ed25519 per RFC 9421, and `derive_bot_auth_public_key(seed)` returns the public key JWK and key id so the key can be published.

### The web_fetch tool

`url` is the only required parameter of `web_fetch` (display name "Web Fetch"). `method` is `GET` or `HEAD`, case-insensitive, default `GET`; other methods are rejected, as is every scheme other than `http://` and `https://` (`file://`, for example), before any network I/O. `as_text: true` converts a page to plain text and `as_markdown: true` converts it to Markdown; otherwise pages come back as raw HTML, and the response `format` field reports `raw`, `text`, or `markdown`. `content_focus` selects `full`, `main`, `readable`, or `agent`, where `agent` is FetchKit's lowest-noise extraction. `crawl: true` discovers and fetches a bounded set of same-origin pages from the seed URL, with `max_pages` capping the count including the seed (default 5, maximum 20). `if_none_match` takes an ETag and `if_modified_since` a Last-Modified value for conditional requests. `save_to_file` appears in the schema only when download is enabled, and there is no `render` parameter.

Responses include `url`, `status_code`, `content_type`, `size`, `format`, `etag`, and `last_modified` alongside `content`. A `HEAD` request returns `method: "HEAD"` with headers and size and no `content`. HTTP 404 and 500 responses come back as successful tool results with their status and body. Inline binary content such as an image or PDF returns metadata plus an error saying only textual content can be fetched. Timeouts are built in at 1 s to first byte and 30 s for the body, with partial content returned when the body timeout is hit. JavaScript rendering is intentionally not bundled; pages that build content client-side need a browser capability such as Browserless.

With download enabled, `save_to_file` writes the response to a path in the session workspace and returns `saved_path` and `bytes_written` instead of inline `content`. Relative paths resolve under the workspace root, and binary downloads round-trip as base64 even though inline binary responses are rejected. A destination that is empty, resolves to the workspace root, or is an existing directory is rejected before anything is written. With download disabled, any `save_to_file` fails with "File download is disabled for this capability" before any network or file activity.

### Fetch egress policy

Every HTTP hop the capability makes, including each redirect hop, crosses the host egress boundary. FetchKit keeps its own pipeline and only the transport is swapped, so a redirect to a forbidden host is denied on its own hop and the final URL after redirects is reported.

SSRF protection is on without any configuration, blocking loopback, RFC 1918, link-local (cloud metadata), CGNAT, and other reserved ranges with DNS pinning to prevent rebinding; the resolved addresses are pinned into the egress request so the connection goes to the vetted IPs. A per-session network access list on the tool context restricts fetches further; a disallowed URL fails with "URL blocked by network access policy: {url}" and no request leaves. An operator-defined, host-wide system allowlist loaded from the environment restricts the agent to permitted public resources with the distinct error "Endpoint blocked by system policy: ...", and when both policies deny, the system-policy error wins. When a system allowlist is active, redirects are pinned to the preflighted host so a cross-host redirect cannot leak the request. Passing `None` for the network access list relies solely on the system allowlist, and `NetworkAccessList::allow_only(["docs.example.test"])` restricts fetches to named hosts.

## Web search: DuckDuckGo instant answers and Brave Search

### DuckDuckGo instant answers

````toml
[dependencies]
everruns = { version = "0.20.0", features = ["duckduckgo"] }
````

````rust
use everruns::prelude::*;

// DuckDuckGo instant answers take no API key or credential.
let agent = Agent::builder()
    .instructions("Answer quick factual questions; say when you are unsure.")
    .model(Model::simulated("42"))
    .capability(DuckDuckGo)
    .build()?;
````

Direct answers, calculations, dictionary definitions, and Wikipedia-style abstracts come back without an API key or credentials through the `duckduckgo` capability, which the zero-config `DuckDuckGo` marker attaches; there are no config keys. It is not a full web or SERP search. The capability is experimental, displayed as "[Experimental] DuckDuckGo" and available in dev mode only, and it may change.

`duckduckgo_instant_answer` is the single tool, and `query` (string) is its only required parameter; a missing or empty query returns "Missing required parameter: query". `no_html` (boolean, default `true`) strips HTML from result text and is the only tunable; region, safe-search, time-range, and result-count parameters are not exposed. The result always includes the original `query` and a `type` classification of `article`, `disambiguation`, `category`, `name`, `exclusive`, or `nothing`. Non-empty sections are added as `heading`, `abstract` and `definition` (each with `text` plus a `source` and `url`), `answer` (`text` and `type`; the query `6*7` yields text `42` and type `calc`), `related_topics` (a flat `{text, url}` list with nested groups flattened, capped at 10), and `results` (official sites as `{text, url}`).

When no instant answer was found the result includes a `note` telling the model the empty result is not definitive and to prefer a web-search or web-fetch tool. The capability's system prompt addition says the same, so the model treats this tool as a lightweight fallback rather than proof that no web pages match. Outside an agent, the client can be called directly for a free-text query and pointed at a custom base URL for proxies or mocks.

### Brave Search

````toml
[dependencies]
everruns = { version = "0.20.0", features = ["web-fetch"] }
everruns-integrations-brave-search = { version = "0.18.2", default-features = false }
````

````rust
use everruns::{Agent, Model};
use everruns_integrations_brave_search::BraveSearch;

// from_env() reads BRAVE_SEARCH_API_KEY and fails here if it is missing.
let agent = Agent::builder()
    .instructions("Search and cite primary sources.")
    .model(Model::simulated("Ready."))
    .capability(BraveSearch::from_env()?)
    .build()?;
````

Brave Search is not a feature flag on `everruns`. You depend on `everruns-integrations-brave-search` directly and pass `BraveSearch::from_env()?` to the builder. `from_env()` reads `BRAVE_SEARCH_API_KEY` at construction and returns a `Result`, so a missing key fails fast at startup rather than in the middle of a turn. `BraveSearch::new` accepts an explicit application-owned key instead. Either way the key is held privately by the client and never placed in capability JSON. Depending on the crate with `default-features = false` omits Platform connector registration, which a Framework application does not need.

Brave Search covers what the instant-answer tool does not, full web search and current news, and both can be attached to the same agent so the model picks per task. The `research-agent` example composes Brave Search and web fetch in one builder chain, with a name, instructions, a provider, a model, and `max_iterations(12)`, so the agent can search the web, read the pages it finds, and answer a sourced question while reporting uncertainty.

## Lua sandbox, host API, and code mode

### The lua capability and tool

````toml
[dependencies]
everruns = { version = "0.20.0", features = ["lua"] }
````

````rust
use everruns::prelude::*;

// FEATURE_LUA=true must be set in the process environment.
let agent = Agent::builder()
    .instructions("Use Lua to compute over workspace files.")
    .model(Model::simulated("computed"))
    .capability(FileSystem)
    .capability("lua")
    .build()?;
````

The `lua` capability lets an agent execute Lua 5.4 scripts in a sandboxed VM with no host or network access. Enabling the `lua` Cargo feature is the first of two switches; the second is `FEATURE_LUA=true` in the environment at runtime, without which the host registers neither `lua` nor `lua_code_mode`. There is no marker type; the capability is attached by its string id. It depends on `session_file_system`, so attach `FileSystem` too; without a session filesystem the tool errors "File system not available in this context". The sandbox runs vendored Lua 5.4, never LuaJIT, and all hardening is on by default with no configuration options to weaken it. In the hosted product the capability is admin-gated.

Scripts arrive through the `lua` tool's required `script` string. `timeout_ms` defaults to 30000 and is capped at 60000, `output` defaults to `auto` and behaves like the bash tool's, and `working_dir` is informational. A successful call returns `stdout` (captured `print` output), `result` (the script's JSON-serialized return value, absent when nil), `success`, `truncated`, and `total_lines`, and `print(...)` output also streams live as `tool.output.delta` events. A runtime error returns a tool error prefixed `Lua error:` with any captured output appended, and a timeout returns "Lua execution timed out after {n}ms". Hard limits are 32 MiB of VM memory, 50,000,000 instructions (a `while true do end` loop terminates with "exceeded instruction budget"), 64 KiB of captured output, and the per-call timeout.

The safe standard library subset is `string`, `table`, `math`, `os`, and `utf8`, including `os.time` and `os.date`. `io`, `package`, `require`, `dofile`, `loadfile`, `load`, `loadstring`, and `collectgarbage` are removed, as are `os.execute`, `os.getenv`, `os.exit`, `os.remove`, `os.rename`, `os.tmpname`, `os.setlocale`, and `string.dump`.

### The host API inside scripts

````lua
local hits = fs.grep("TODO", "/workspace/src")
local report = { count = #hits, files = {} }
for _, hit in ipairs(hits) do
  report.files[#report.files + 1] = hit.path .. ":" .. hit.line_number
end
fs.write("/workspace/todo-report.json", json.encode(report))
print("todos found:", #hits)
return report.count
````

`fs.read(path)` and `fs.write(path, s)` read and write files, and `fs.append(path, s)` treats a missing file as empty, so appending creates it. `fs.exists(path)` is true for files and directories. `fs.list(path)` returns a 1-indexed array of `{name, is_dir, size}` tables and `fs.stat(path)` returns a table or nil. `fs.mkdir(path)` and `fs.remove(path[, recursive])` manage structure. `fs.grep(pattern[, path])` returns an array of `{path, line_number, line}` hits from the store's indexed search, and omitting the path searches the whole workspace. `fs.*` paths default to `/workspace`. `json.encode(value)` and `json.decode(string)` handle JSON, and `base64.encode(string)` and `base64.decode(string)` handle Base64.

`http.get(url)` and `http.post(url, body)` return `{ status, body }`. The `http` global exists only when the host provides an egress service and a non-empty network allow-list; otherwise it is nil. HTTP is fail-closed: a URL outside the allow-list fails with "network egress denied", requests are DNS-pinned and time out at 15 s, POST bodies go as `application/json`, and response bodies are truncated to 1 MiB.

### Code mode

````rust
use everruns::prelude::*;
use serde_json::json;

// `add` is an #[everruns::tool] async fn add(a: f64, b: f64) -> f64 from chapter 03.
let agent = Agent::builder()
    .instructions("Solve tasks by writing one Lua script that calls the tools you need.")
    .model(Model::simulated("done"))
    .capability(FileSystem)
    .capability("lua")
    .capability(CapabilityRef::with_config("lua_code_mode", json!({ "keep_visible": ["stat_file"] })))
    .tool(add())
    .build()?;
````

The `lua_code_mode_agent` example (package `everruns-host`, feature `lua`) shows an agent working the way this builder sets up: the `lua_code_mode` capability (requires `lua`) makes the `lua` tool the agent's primary action surface. It hides non-essential tools from the model's direct tool list and exposes them inside scripts as `tools.<name>(args_table)`, for example `local r = tools.add{ a = 1, b = 2 }`, which returns the tool's JSON result as a Lua table. The `tools` global is nil when no tools are routed. One script can read inputs and call several sibling tools, assembling the result in a single turn instead of a chain of round-trips. Hidden tools are removed only from the model-facing list; the executable registry is untouched, so no tool can become unreachable.

Routing follows a fixed eligibility rule: only tools with auto policy that are neither destructive nor CPU-bound go through Lua. `lua` and `bash` themselves stay direct, as do approval-gated and client-side tools, and a code-mode call cannot recursively open code mode. The model discovers hidden tools from the `lua` tool's description, which is extended with a `tools.<name>{ args }` catalog listing each hidden tool with a compact typed signature (required args as `name: type`, optional as `name?: type`) and the first sentence of its description. Two config keys exist: `keep_visible`, an array of tool names to keep directly callable, and `full_schemas`, a boolean (default `false`) that embeds each hidden tool's complete JSON Schema in the catalog. Unknown keys and wrong types are rejected.

## Attaching MCP servers: the McpServer builder, scoping, discovery, and tool naming

### The McpServer builder

````toml
[dependencies]
everruns = { version = "0.20.0", features = ["mcp-stdio"] }
````

````rust
use everruns::prelude::*;

// The bearer token for the private server comes from the process environment.
let secret = std::env::var("PRIVATE_MCP_TOKEN")?;

let agent = Agent::builder()
    .instructions("Use the catalog and docs servers to answer product questions.")
    .model(Model::simulated("answered"))
    .mcp_server(McpServer::http("catalog", "https://example.com/mcp"))
    .mcp_server(
        McpServer::http("private", "https://private.example/mcp")
            .header("Authorization", format!("Bearer {secret}")),
    )
    .mcp_server(
        McpServer::stdio("docs", "/usr/local/bin/docs-mcp")
            .arg("--root")
            .arg("/srv/docs")
            .env("DOCS_LOCALE", "en"),
    )
    .build()?;
````

The `mcp` feature connects agents to MCP (Model Context Protocol) servers through a transport-agnostic client, so the same agent code works however a server is reached. Each active server becomes a virtual capability whose tools join the agent's tool set and are called like native tools.

A server is added with the builder's `mcp_server` method, which takes an `McpServer` value; calling it more than once attaches multiple servers. `McpServer::http(name, url)` configures a remote Streamable HTTP server. `McpServer::stdio(name, command)`, available only with the `mcp-stdio` feature, launches a local server as a child process speaking JSON-RPC over stdin and stdout; `arg(value)` and `args(iter)` append command-line arguments one at a time or from an iterator, and `env(name, value)` sets an environment variable on the child. On an HTTP server, `header(name, value)` attaches a literal header to every request, which is how a bearer token is supplied, and `tool_discovery(false)` turns off live discovery so the server is connected for execution without its tools being advertised to the model.

Debug-printing an `McpServer` reports shape only: header values, env values, the URL, the command path, and the arguments are omitted, so a server configuration can be logged without leaking credentials.

Validation happens at `build()` and surfaces as `BuildError::InvalidMcpServer { reason }`. A blank name is rejected, as is a name whose sanitized form contains `__`, which is reserved as the server/tool separator. Two servers with the same name are rejected, and so are names that collide after sanitization, such as `github-prod` and `github_prod`, so tool names cannot silently alias. An HTTP server without a URL and a stdio server without a command each fail. Server names may use letters, digits, and single interior underscores; names with trailing or doubled underscores or dashes, such as `docs_` or `docs__private`, are ambiguous tool prefixes, and a host that meets one publishes no tools for it and logs it as ignored.

### Scoping and merging

MCP servers can be declared at three scopes. Agent scope is the builder shown above; harness and session scope use the host builders introduced in chapter 10. The host merges the layers in order, harness, then agent, then session, into one effective set with later layers overriding earlier ones by name, and servers contributed by capabilities are merged with the explicitly declared ones. An agent's servers are inherited by its sessions. Each scoped server has a transport type, URL or command, headers, auth mode, protocol version mode, OAuth provider id, secret bindings, and a tool-discovery flag. Without the `mcp-stdio` feature a stdio server is skipped with the warning "stdio MCP server ignored: runtime built without the `mcp-stdio` feature" rather than failing the session, and a stdio server missing its command is skipped the same way.

### Discovery and tool naming

At the start of a turn the host discovers tools from all servers in parallel, up to 16 at once in stable order, so setup latency is bounded by the slowest server rather than the sum. Discovery results are cached per session with stale-while-revalidate semantics. Fresh entries come from memory and stale entries are served immediately while revalidating in the background; a cold entry blocks on a single `tools/list`. A server that fails discovery is skipped for that turn with a warning, so one unreachable server does not fail the turn. The cache is invalidated whenever a capability is activated or deactivated.

MCP tools reach the model under collision-free names of the form `mcp_{sanitized_server_name}__{tool_name}`. Server `microsoft-learn` with tool `search` becomes `mcp_microsoft_learn__search`: hyphens become underscores and a double underscore separates server from tool. A proxy layer registers them as first-class registry tools, so the model calls them like any native tool rather than through a separate dispatch path.

Each server capability has the id `mcp:{server_uuid}`, where the uuid is derived deterministically from the session and the server name within a session. The capability's display name is the server name and its category is "MCP Servers". MCP tool annotations map to the framework's tool hints: `readOnlyHint`, `destructiveHint`, `idempotentHint`, and `openWorldHint`. MCP tools are treated as open-world by default, and the input schema passes through unchanged.

## MCP client, HTTP and stdio transports, protocol eras, and tool result mapping

### Transports

An MCP endpoint is either HTTP or stdio. An HTTP endpoint is a URL plus per-server headers, and the HTTP transport, built with `HttpTransport::new(egress)` around the egress service, is always compiled; the two-method `McpTransport` trait (`list_tools` and `call_tool`) is the extension point for a custom transport such as WebSocket or an in-process test double. A stdio endpoint is a command with its arguments and environment, and exists only with the stdio feature. The stdio transport spawns a fresh server process for every tool listing and every tool call: spawn, `initialize` handshake at protocol version `2024-11-05`, `notifications/initialized`, one request, tear down. No state leaks between invocations. The whole call is bounded by a fixed 60 s timeout. Only stdin and stdout are piped and stderr is discarded; the child inherits the host's working directory. Credentials from the auth provider are not used for stdio, so any authentication comes through the process environment set with `env(name, value)` on the builder.

````rust
use everruns::prelude::*;

// The echo server takes no credential; a stdio server that needs one reads it
// from the child environment set with .env(name, value).
let agent = Agent::builder()
    .instructions("Echo what the user says through the echo server.")
    .model(Model::simulated("echoed"))
    .mcp_server(McpServer::stdio("echo", "target/debug/mcp_stdio_echo"))
    .build()?;
````

The bundled `mcp_stdio_echo` binary (package `everruns-mcp`, feature `stdio`) is a minimal MCP stdio server for smoke-testing the transport. It exposes one tool, `echo`, whose input schema requires a string `message`, and returns the message as text content; attached under the name `echo`, it reaches the model as `mcp_echo__echo`.

### Client operations for advanced hosts

````rust
use std::sync::Arc;
use everruns_mcp::{McpClient, McpConnection, NoAuthProvider};

// `egress` is the host's Arc<dyn EgressService>. NoAuthProvider sends no
// credential; this connection is for a server that needs none.
let client = McpClient::new(egress, Arc::new(NoAuthProvider));
let connection = McpConnection::http("docs", "https://example.com/mcp");
let tools = client.discover(&connection).await?;
let result = client.call(&connection, "search", serde_json::json!({ "q": "sessions" })).await?;
````

`McpClient::new(egress, auth)` builds the client over an egress service and a pluggable auth provider, so all outbound HTTP passes through the caller's egress boundary. `McpConnection::http(name, url)` creates a no-auth connection with protocol mode `Auto` and empty headers. `discover` issues `tools/list` and `call` issues `tools/call`, returning the raw MCP result; `call_as_tool_result(connection, tool_call_id, tool_name, arguments)` returns the result already mapped into the framework's `ToolResult`. An `McpExecutor::new(client, resolver)` pairs a shared client with a connection resolver to execute `mcp_*` tool calls end to end; `StaticConnectionResolver` holds a fixed set of connections keyed by sanitized server name for hosts that know their servers up front, and the `McpConnectionResolver` trait accepts a custom strategy. Stateless free functions such as `http_list_tools` serve code that only holds an egress service.

### Protocol eras and negotiation

The client speaks three MCP protocol eras through one code path. `2025-03-26` and `2025-06-18` are stateful: the client runs the `initialize` handshake and sends `notifications/initialized`, echoing any `Mcp-Session-Id` it receives on later requests. `2026-07-28` is stateless: there is no handshake; the protocol version and client info are sent in `_meta` on every request, and the routable headers `MCP-Protocol-Version`, `Mcp-Method`, and `Mcp-Name` let edge infrastructure route without parsing the body.

The default protocol mode is `Auto`. The client starts stateless, and if the server's reply looks like a handshake is required, it performs the stateful handshake and retries once. The outcome is cached for 300 s per URL, server name, mode, headers, and credential, so different auth contexts never share a session. If both eras fail, the error reports both attempts. A server can be pinned to one era per connection with `McpConnection::with_protocol_mode(mode)` set to `V2026July`, `V2025June`, or `V2025March`; pinned stateful modes handshake up front instead of probing. The client declares which mid-call input types it can answer through `ClientCapabilities`, and by default it declares none; URL-mode elicitation is declared only when a URL elicitation handler is installed. Tool discovery times out after 30 s and tool execution after 60 s.

Every outbound MCP request passes DNS-pinned SSRF validation, and a server URL resolving to a blocked address is rejected before any bytes are sent. All MCP HTTP traffic, including OAuth requests, is tagged as MCP traffic at the egress boundary. A server's `tools/list` result is cached for the server-declared `ttlMs`; servers that send no caching hints are never cached, and results marked `cacheScope: "private"` are keyed by credential so they never cross users.

### Tool result mapping

`map_tool_call_result(&result)` converts an MCP `tools/call` result into a model-ready JSON value plus a separate list of images. Text parts become `{"result": "..."}`, joined with newlines when there are several. Images become native image content blocks rather than embedded JSON. Resource parts switch the output to a `{"content": [...]}` list of typed parts, and an error result becomes `{"error": "..."}` with only the text parts kept. Responses framed as SSE (`event:` and `data:` lines) are unwrapped automatically.

## MCP authentication, OAuth 2.1, and credential bindings

### Auth providers and credentials

````rust
use std::sync::Arc;
use everruns_mcp::{McpClient, McpCredential, StaticAuthProvider};

// Both tokens are read from the process environment at startup.
let auth = StaticAuthProvider::new()
    .with_bearer("docs", std::env::var("DOCS_MCP_TOKEN")?)
    .with_credential(
        "legacy",
        McpCredential::authorization(format!("Basic {}", std::env::var("LEGACY_MCP_BASIC")?)),
    );

let client = McpClient::new(egress, Arc::new(auth));
````

How an agent authenticates to an MCP server is set per server by an auth mode of none, API key, or OAuth and, for OAuth, a reference to an OAuth provider by id. Credential acquisition is an injectable strategy: the host implements the `McpAuthProvider` trait's one method, `authorization`, which receives an `McpAuthRequest` with the logical `server_name`, the configured `auth_mode`, and an optional `oauth_provider_id`.

`McpCredential::bearer(token)` formats the `Authorization` header as `Bearer <token>`, and `McpCredential::authorization(value)` supplies a verbatim header value for `Basic ...` or a vendor scheme. `StaticAuthProvider` holds long-lived tokens for multiple servers keyed by logical server name; unregistered servers fall through as unauthenticated. `NoAuthProvider` is the default and always returns no credential.

### OAuth 2.1 login

````rust
use everruns_mcp::oauth::{complete_login, prepare_login};

// `egress` is the host's Arc<dyn EgressService>. No client secret is configured
// here: the server's registration endpoint issues a public client on the spot.
let prepared = prepare_login(
    egress.as_ref(),
    "https://mcp.example.com/mcp",
    "http://127.0.0.1:1455/callback",
    "my-cli",
    None,
    Some("mcp:read"),
).await?;
open_browser(&prepared.authorization_url);

// The loopback listener receives ?code=...&state=...&iss=...
let tokens = complete_login(egress.as_ref(), &prepared, &code, &state, iss.as_deref()).await?;
````

An OAuth-protected remote MCP server is handled by an MCP-specific OAuth 2.1 flow you do not write: discovery, dynamic registration, PKCE, authorization, code exchange, and refresh. The login is two calls around one browser round trip. `prepare_login` takes the egress service, server URL, redirect URI, client name, an optional pre-registered `RegisteredClient`, and an optional scope, and returns a `PreparedLogin` holding the `authorization_url` to send the user-agent to, the CSRF `state`, the `issuer`, and the bound `resource`. How the code comes back is up to the host. The `PreparedLogin` contains the PKCE verifier and must not be logged or persisted unencrypted.

`prepare_login` fetches the server's RFC 9728 protected-resource metadata (a server that publishes none is not an error; its origin becomes the issuer), validates that the resource uses HTTPS and shares the server's origin, and discovers authorization-server metadata. If no client was supplied and the server advertises a registration endpoint, it dynamically registers a public client (RFC 7591). It then generates a fresh S256 PKCE pair and builds the authorization URL, including the RFC 8707 `resource` indicator so a token for one server cannot be replayed against another.

`complete_login` enforces the CSRF check and RFC 9207 issuer validation when the callback includes `iss`, then exchanges the code. The resulting `TokenSet` holds the access token, optional refresh token, token type (default `Bearer`), absolute expiry, granted scope, and the token endpoint and client credentials needed to refresh in one self-contained call. It serializes for encrypted persistence only. Events and snapshots never receive it, and neither do API responses. All OAuth endpoints must be absolute HTTPS, with plaintext HTTP permitted only on loopback hosts so native-app redirect URIs such as `http://127.0.0.1:1455/callback` work. Failures are a typed `OAuthError` set labeled with the step that failed.

### Keeping tokens fresh

````rust
use std::sync::Arc;
use everruns_mcp::McpClient;
use everruns_mcp::oauth::{InMemoryTokenStore, OAuthAuthProvider};

// `tokens` is the TokenSet returned by complete_login above.
let store = InMemoryTokenStore::seeded("docs", tokens);
let auth = OAuthAuthProvider::new(store, egress.clone()).refresh_skew_seconds(120);
let client = McpClient::new(egress, Arc::new(auth));
````

Token storage is pluggable through the two-method `McpTokenStore` trait, `load` and `save` by logical server name; a control plane writes encrypted rows and a CLI writes a file. `InMemoryTokenStore` is built in for tests or hosts that can re-authorize on every restart. `OAuthAuthProvider::new(store, egress)` wraps a store and serves as the client's auth provider. A live token is returned with no network call, a token near expiry is refreshed and saved before use, and refreshes are serialized so a token is renewed once rather than once per concurrent tool call. The refresh skew defaults to 60 s and is tuned with `refresh_skew_seconds`. When a server needs an OAuth connection that has not been made yet, the executor short-circuits the tool call into a connection-required result so the host renders an inline connect prompt instead of a raw 401.

### Credential bindings

A write-only secret can be bound to a specific MCP server, tool, and input parameter on an agent, for example a `channel_key` parameter of a `visti_send` tool. The MCP server capability must be attached to the agent first. The bound parameter is stripped from the model-visible tool schema and the decrypted value is injected only when the MCP request is sent, so it never appears in chat, instructions, memory, or session storage. The same binding works for triggers that create a new session per invocation, which distinguishes it from the single-session `secret_store` covered in chapter 07.

If the model tries to supply a bound parameter itself, the call fails with code `credential_override_rejected`. If a binding has no configured value, the call returns code `credential_required` with a `setup_url` and `credential_label`. Injected credentials are scrubbed from the tool result and from transport error text before anything reaches the model, events, persistence, or logs; the replacement text is `[REDACTED MCP CREDENTIAL]`.

## MCP URL elicitation and human consent

### What a URL elicitation is

A `2026-07-28` MCP server can pause a tool call to ask a human to visit a URL out of band: to authorize a third party, paste an API key on the server's own page, approve a charge, or click a consent screen. The server answers `tools/call` with an `input_required` result that contains an `elicitation/create` request in `mode: "url"`. The secret entered on that page never transits the client and never enters model context. The client never answers the elicitation itself; it collects a human's decision and, on consent, retries the tool call with the server's echoed request state under a fresh JSON-RPC id. At most two rounds are attempted; after that the error tells the user to finish the browser step and run the tool again.

Server-supplied elicitation URLs are validated before any handler or user sees them. Only `https` is accepted, with `http` allowed solely on loopback for local development; `javascript:`, `file:`, `data:`, and non-URLs are rejected. If a server requests an input type the client never declared, the failure names the offending server and tool rather than returning a silent empty result.

### Handlers

````rust
use std::sync::Arc;
use async_trait::async_trait;
use everruns_mcp::{
    ElicitationAction, McpClient, NoAuthProvider, UrlElicitation, UrlElicitationHandler,
};

struct TerminalConsent;

#[async_trait]
impl UrlElicitationHandler for TerminalConsent {
    async fn request_url_consent(&self, e: &UrlElicitation) -> anyhow::Result<ElicitationAction> {
        println!("{} needs you to open {} before '{}' can run.", e.server_name, e.url, e.tool_name);
        if e.punycode {
            println!("warning: host {} contains an internationalized (xn--) label", e.host);
        }
        Ok(if ask_yes_no("Open it in your browser?") { ElicitationAction::Accept } else { ElicitationAction::Decline })
    }
}

// NoAuthProvider: the server itself collects its secret on the elicited page,
// so the client sends no credential.
let client = McpClient::with_url_elicitation(egress, Arc::new(NoAuthProvider), Arc::new(TerminalConsent));
````

A host plugs in its consent surface, such as a chat card or a CLI prompt, by implementing the one method of `UrlElicitationHandler` and passing it to `McpClient::with_url_elicitation`. The handler must show the full URL, must not fetch it, must not open it without explicit consent, and should return `Cancel` promptly rather than block when no human is reachable. The `UrlElicitation` it receives holds `server_name`, `tool_name`, the server's request `key`, a `message`, the `url`, the pre-extracted `host` so the surface can highlight the domain against subdomain spoofing, and a `punycode` flag. `Accept` means the human consented to opening the URL, not that the out-of-band step finished.

Three built-in handlers cover common hosts. `DeclineUrlElicitations` is for unattended runs that still need the elicitation capability declared but have no reachable human; it declines every request. `RelayUrlElicitations` is for hosts that can show the URL but cannot block on the user; it always answers cancel, and the in-process runtime wires this one by default. `ConsentingUrlElicitations::new(store)` is for hosts that can pause a turn and resume it after collecting consent out of band; the retry answers accept once a recorded consent exists. Installing no handler, by building the client with `McpClient::new`, is different from installing the decline handler: with no handler the client declares no elicitation capability, which under the protocol forbids servers from asking.

### Consent storage

The consenting handler reads consents from a pluggable `ElicitationConsentStore` whose one method, `take_consent(server, tool)`, takes and consumes a consent. Consent has to outlive the process that asked for it, which is why it is a store rather than a channel. A `StoredConsent` captures the server, tool, the host the user saw, and an expiry, and holds no secret. Consents expire after `CONSENT_TTL`, 30 minutes. One consent authorizes exactly one accept. A server that elicits a different domain than the user consented to is asked again, and a store that cannot be read fails closed. Storage keys have the form `mcp/elicitation-consent/{server}/{tool}`.

### What the application sees

A pending elicitation reaches the application as a structured `url_elicitation_required` tool result with the `url`, `url_host`, `url_is_punycode`, `server`, `tool`, the `retry_tool` name the model can call again after the human finishes, a `message`, and a `declined` flag. When the human refused, `declined` is true and nothing more is asked. The engine's URL elicitation hook turns such a result into a synthetic `confirm_url_elicitation` tool call so a client can render a consent card, and the turn pauses waiting for the decision, the act-phase pause chapter 05 introduced. Clients that cannot render the card keep the older behavior in which the elicitation is relayed as an ordinary tool result and the user re-runs the tool. The pause is swept after `TOOL_RESULT_TIMEOUT_SECS` (default 300 s).

The `mcp-url-elicitation` example ships a Node stub server that refuses to run its `run_revenue_report` tool until a human enters an API key on the server's `/connect` page. Because the built-in SSRF checks block `localhost` and private ranges, the stub must be exposed through a tunnel with `PUBLIC_URL` set. The final step greps the session events for the key and expects no match.

## Network access, egress selection, and risk gates across capabilities

Every network-capable capability in this chapter sends its traffic through the same host egress boundary, and two policies apply there regardless of which capability produced the request. The network access list can be declared at the harness, agent, and session layers and is merged as chapter 05 described: layering can only narrow. The merged list is enforced for every hop of every outbound request made by the shell, web fetch, Lua HTTP, and MCP. The deployment-wide system allowlist, toggled by an environment variable whose name is held in the `SYSTEM_ALLOWLIST_ENABLED_ENV` constant, restricts outbound destinations independently of per-agent lists and applies to shell traffic and web fetch alike.

````toml
[dependencies]
# Offline: no HTTP client is linked, and the egress service is the disabled one.
everruns = "0.20.0"

# Any one of these switches the host to direct egress configured from the environment.
# everruns = { version = "0.20.0", features = ["mcp"] }
````

The same feature flags also pick the egress service. A host with no network-capable feature enabled remains fully offline and does not link the concrete HTTP client crate; its egress service is `DisabledEgressService`. Enabling any of `bashkit`, `web-fetch`, `duckduckgo`, `lua`, or `mcp` switches the host to a `DirectEgressService` configured from the environment. `mcp` alone switches egress on even though it registers no capability by itself; the MCP servers you attach are what use it. Advanced hosts obtain the matching service with `runtime_egress_service()`, the companion to the `runtime_capability_registry()` call from the first section, and its result reflects the same feature selection.

Bashkit shell, web fetch, Lua, and Lua code mode all carry a high risk rating. In the hosted product that rating puts them behind role, network-access, and runtime feature gates, and the gate applies to new assignments only.

*2026-09-17 03:24 - claude-fable-5.1*
