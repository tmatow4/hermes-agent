# Hermes Core Architecture Report

Prepared on branch `tmatow4/hermes-core-architecture-research`.

This report explains the implementation shape behind the agent loop,
subagents, context engineering, memory, prompts, orchestration, and tools. It is
intended for a developer who wants to understand why Hermes feels more like an
agent runtime than a simple chat wrapper.

## Executive Summary

Hermes is centered on one large synchronous runtime class:
`AIAgent` in `run_agent.py`. Most user-facing surfaces - CLI, TUI, gateway,
batch runner, API/ACP adapters, cron, and plugins - eventually feed a prompt
into an `AIAgent` instance and receive a final response plus a durable message
trace.

The power of the system comes from several layers working together:

1. A provider-agnostic agent loop that normalizes OpenAI-compatible chat,
   OpenAI Responses/Codex, Anthropic Messages, Bedrock Converse, and related
   transports into the same internal message/tool model.
2. A self-registering tool registry and toolset layer that let built-in tools,
   MCP/plugin tools, platform tools, and agent-only tools coexist.
3. A cache-friendly prompt architecture: stable identity/tool guidance first,
   project context next, and session metadata/memory last. The full prompt is
   stored and reused to preserve upstream prompt cache behavior.
4. Multiple context-control mechanisms: context files, lazy subdirectory hints,
   oversized tool result persistence, per-turn tool budget enforcement,
   compression engines, session search, and memory provider prefetch.
5. Subagents implemented as real child `AIAgent` instances with isolated
   prompts, restricted tools, separate budgets, independent execution, and
   parent-visible summaries.
6. A plugin system that can register tools, hooks, CLI commands, platform
   backends, and context engines without hardcoding plugin-specific logic into
   core files.
7. Harness paths for scaled work and training data: batch execution,
   trajectories, and `execute_code` programmatic tool calling.

Put differently: Hermes keeps the core loop simple enough to reason about
while building a lot of operational machinery around it to preserve context,
control side effects, support multiple providers, and let work fan out.

## Core Flow

```text
User surface
  CLI / TUI / gateway / cron / API / batch
        |
        v
run_agent.AIAgent
  - resolve provider and API mode
  - load tools and toolsets
  - initialize memory and context engine
  - build or reuse cached system prompt
        |
        v
Main loop
  build API messages
  call provider transport
  normalize response
  if tool calls: execute tools, append results, maybe compact, continue
  if final text: persist, sync memory, run hooks, return
        |
        v
Persistence and side systems
  state.db sessions, memory providers, plugin hooks, trajectories,
  tool environments, subagents, gateway callbacks
```

The main design idea is that almost everything becomes a message, a tool, a
hook, or a context source. That common shape lets CLI sessions, messaging
platforms, batch prompts, and subagents share the same operational semantics.

## Entry Points

`run_agent.py` is the runtime center, but it is not the only important entry
point:

- `cli.py` owns the classic interactive CLI and slash command dispatch.
- `ui-tui/` owns the Ink-based terminal UI, while `tui_gateway/` hosts the
  Python JSON-RPC backend that runs the real agent.
- `gateway/run.py` adapts messaging platforms and routes platform events into
  Hermes sessions.
- `batch_runner.py` runs many prompts in parallel worker processes and saves
  trajectories.
- `acp_adapter/` exposes Hermes to editor integrations.
- `hermes_cli/plugins.py` discovers plugin tools, hooks, platform adapters, and
  context engines.

The important architecture point is that these surfaces are wrappers. They may
add callbacks, session metadata, streaming, approval handling, or platform
rendering rules, but the central conversation logic remains `AIAgent`.

## AIAgent Construction

`AIAgent.__init__` takes a large number of parameters because it is the meeting
point for provider routing, tool access, session state, callbacks, budgets,
credentials, compression, memory, and platform behavior.

Key initialization responsibilities:

- Resolve API mode from explicit config, provider, and base URL. Supported
  modes include `chat_completions`, `codex_responses`, `anthropic_messages`,
  `bedrock_converse`, and `codex_app_server`.
- Upgrade GPT/Codex-style OpenAI requests to the Responses path when the
  provider/model requires it, while excluding incompatible runtimes such as
  Azure OpenAI and ACP routes.
- Load tool schemas via `model_tools.get_tool_definitions`.
- Record `valid_tool_names` so prompt assembly can inject only relevant tool
  guidance.
- Initialize the built-in file-backed memory store unless memory is skipped.
- Initialize at most one external memory provider through `MemoryManager`.
- Initialize the selected context engine. The built-in default is
  `ContextCompressor`; plugins can provide alternatives.
- Set up callbacks for streaming, reasoning, tool progress, approvals,
  interruption, gateway/TUI status, and file mutation tracking.
- Create or reuse the SQLite session store.

This makes `AIAgent` heavy, but it also makes it the only component that knows
the full runtime state. Subagents and batch workers benefit from this because
they can instantiate the same runtime with different constraints instead of
using a separate miniature agent implementation.

## Main Agent Loop

The main loop lives in `AIAgent.run_conversation`. It is synchronous by design:
the loop repeatedly calls a model, handles tool calls, appends results, and
continues until final text, interrupt, guardrail halt, or iteration exhaustion.

The high-level turn sequence is:

1. Ensure the session row exists and establish runtime session context.
2. Sanitize and append the new user message.
3. Build or reuse the stored system prompt.
4. Run preflight compression if the message history plus system prompt plus tool
   schemas is already too large.
5. Invoke plugin `pre_llm_call` hooks. Returned context is injected into the
   current user message rather than the cached system prompt.
6. Each iteration builds a provider-facing message copy, preserving the internal
   durable history.
7. The provider call happens through the active transport. Streaming paths feed
   callbacks, but return a normalized response object so the rest of the loop
   stays unified.
8. If the model returned tool calls, Hermes validates tool names and JSON,
   executes tools, appends `role=tool` messages, enforces tool-result budgets,
   checks compression, and loops.
9. If the model returned final content, Hermes persists messages, updates usage,
   runs output hooks, queues memory sync/prefetch, and returns.
10. If iterations are exhausted, Hermes makes a toolless summary call so the
    user gets a coherent stop state rather than a silent cutoff.

The loop also contains a lot of survival logic: fallback models, context-length
retries, malformed tool-call repairs, orphaned tool-result sanitization,
thinking-only message cleanup, reasoning field preservation, interrupted tool
batch handling, and retry behavior for provider-specific quirks.

## Provider Normalization

Hermes tries to keep provider differences at the edge:

- `agent/codex_responses_adapter.py` converts internal chat-style messages into
  OpenAI Responses input items and preserves Responses-specific call IDs,
  assistant message items, and encrypted reasoning blobs.
- Anthropic and Bedrock adapters translate the same internal message stream into
  their native wire formats.
- `AIAgent._build_assistant_message` captures provider-specific reasoning
  fields back into the internal message record.
- Provider-facing cleanup runs on a copy of messages. The stored session can
  keep richer fields while strict APIs receive only fields they accept.

This is why the internal message shape is so important. The loop stores as much
as it safely can, then projects it into the target provider's schema at call
time.

## System Prompt Design

Prompt assembly is split between `agent/prompt_builder.py` and
`AIAgent._build_system_prompt_parts`.

The prompt has three conceptual tiers:

- Stable: identity, Hermes help guidance, tool-specific guidance, skills
  guidance, model-family operational guidance, environment hints, and platform
  hints.
- Context: caller-supplied system message plus project context files from the
  working directory.
- Volatile: built-in memory snapshot, external memory provider block, timestamp,
  session ID, model, and provider.

The implementation then joins all three tiers into one full system prompt and
caches it for the agent session. On resume, Hermes can reuse the exact stored
system prompt from `state.db`. This is deliberate: a stable prefix helps prompt
caches and avoids subtle behavior changes mid-session.

Important nuance: the code labels memory/timestamp as "volatile", but the full
system prompt is still cached for the lifetime of the agent instance and is only
rebuilt after compression. Built-in memory writes update disk immediately, but
the current session's injected memory block remains the snapshot captured at
prompt build time.

Project context loading follows a priority order:

1. `.hermes.md` or `HERMES.md`, discovered by walking up to the git root.
2. `AGENTS.md` or `agents.md`, current working directory only.
3. `CLAUDE.md` or `claude.md`, current working directory only.
4. `.cursorrules` or `.cursor/rules/*.mdc`, current working directory only.

Each loaded context file is scanned for prompt-injection patterns and truncated
with a head/tail strategy if it is too large.

## Context Engineering

Hermes has several independent context controls. They complement each other
rather than replacing one another.

Startup context files give the model repo or project rules at session start.

Subdirectory hints in `agent/subdirectory_hints.py` discover additional
`AGENTS.md`, `CLAUDE.md`, or `.cursorrules` files as tool calls touch deeper
paths. These hints are appended to tool results, not inserted into the system
prompt, so the cached prompt stays stable.

Large tool result persistence in `tools/tool_result_storage.py` prevents big
outputs from flooding the conversation. Oversized results are written into the
active sandbox or backend temp directory, and the model receives a preview plus
a path it can read with file tools. A second aggregate budget pass catches turns
where many medium outputs would otherwise exceed the per-turn budget.

The context engine abstraction in `agent/context_engine.py` defines a lifecycle
for compression and context management:

- initialize and session start
- optional tool registration
- response updates
- `should_compress`
- `compress`
- session end

The default engine is `ContextCompressor` in `agent/context_compressor.py`. It
estimates token pressure, prunes old tool results, protects recent tail context,
summarizes the middle, preserves role/tool-call validity, and inserts a
structured summary. Compression also splits the session in SQLite using
`parent_session_id`, so long conversations remain resumable and searchable.

Session search is a separate recall path. `tools/session_search_tool.py` uses
SQLite FTS5 to find prior sessions, then uses an auxiliary model to summarize
the relevant transcript window. This keeps old raw transcripts out of the main
context unless the model explicitly asks for them.

External memory providers add another live-context path. `MemoryManager` can
prefetch context for the current user message and inject it into that message as
a fenced memory context block. This is distinct from the provider's static
system prompt block.

## Memory Architecture

Hermes has three memory-like systems with different jobs:

1. Built-in memory files:
   - `tools/memory_tool.py`
   - `MEMORY.md` for durable facts
   - `USER.md` for user profile facts
   - bounded, file-backed, injection-scanned, atomic writes
   - loaded into a frozen prompt snapshot at session start

2. External memory providers:
   - `agent/memory_provider.py`
   - `agent/memory_manager.py`
   - one active provider at a time
   - can expose tools, return system prompt blocks, prefetch context, sync turns,
     receive compression/session/delegation hooks, and handle provider-specific
     writes

3. Session database recall:
   - `hermes_state.py`
   - SQLite `state.db`, WAL with fallback, message history, FTS5 and trigram FTS
   - used by resume, history, titles, session search, and compression lineage

The separation matters. Built-in memory is for compact durable facts. External
memory is for provider-backed semantic recall or richer identity systems.
Session search is for prior transcripts and task history. Mixing those concerns
would make memory stale, verbose, and harder to trust.

## Tool Architecture

Tool orchestration has three main layers:

- `tools/registry.py` is the self-registering registry. Tool files call
  `registry.register(...)` at import time. Discovery scans `tools/*.py` for
  files that contain top-level registrations.
- `toolsets.py` defines named tool bundles and the default core tool list. A
  tool may be registered but still unavailable unless it is exposed through a
  selected toolset.
- `model_tools.py` builds provider-facing tool schemas from enabled/disabled
  toolsets, registry availability checks, plugin tools, and dynamic schema
  overrides.

There are also agent-level tools that the registry advertises but the main
agent loop intercepts before normal dispatch:

- `todo`
- `memory`
- `session_search`
- `delegate_task`

These need agent/session context, so `run_agent.py` handles them directly or
routes them to specialized managers.

Tool execution supports both sequential and concurrent paths. Hermes only
parallelizes when the batch is safe: no explicitly interactive or serial tools,
JSON arguments parse cleanly, path-scoped tools do not overlap, and mutating MCP
tools are only parallelized when the server opts in. This lets read-heavy model
responses run faster without turning side effects into a race.

The tool loop is guarded by:

- pre/post plugin hooks
- approval callbacks for risky commands
- repeated-failure/no-progress guardrails
- output size persistence
- per-turn aggregate budget enforcement
- interrupted-batch skip messages
- subdirectory context discovery
- file mutation verification

This is why "tool calling" in Hermes is not just dispatching a Python function.
The runtime wraps tool calls with context, safety, persistence, observability,
and retry controls.

## Programmatic Tool Calling

`tools/code_execution_tool.py` implements `execute_code`, described there as
"Programmatic Tool Calling". The model writes a Python script; the script calls
a generated `hermes_tools.py` module; those calls are proxied back to the parent
Hermes process over RPC.

There are two transports:

- Local backend: Unix domain socket or loopback TCP, parent RPC listener, child
  Python process.
- Remote backend: file-based RPC in the active terminal backend, such as Docker,
  SSH, Modal, Daytona, or another environment.

Only the script's stdout returns to the model. Intermediate tool results do not
enter the LLM context window. This is one of the strongest architectural moves
in Hermes: multi-step file/search/read/write workflows can collapse into one
model iteration while still using normal Hermes tools under parent supervision.

`run_agent.py` refunds the iteration budget when the only tool call in a turn is
`execute_code`, reflecting that it is more like a local harness expansion than a
normal model-to-tool round.

## Subagents And Delegation

Subagents are implemented in `tools/delegate_tool.py`. They are not a separate
mini-runtime. Each child is another `AIAgent` instance with:

- a focused child system prompt passed as `ephemeral_system_prompt`
- `skip_context_files=True`
- `skip_memory=True`
- a parent session ID
- restricted toolsets inherited from the parent
- blocked tools such as `clarify`, `memory`, `send_message`, and nested
  `delegate_task` unless the role is explicitly `orchestrator`
- its own iteration budget
- quiet mode and parent callbacks for progress/interrupts

The parent sees only structured child results. In batch delegation, multiple
children run through a `ThreadPoolExecutor` with a max concurrency cap. The
parent polls futures so interrupts can abort waiting. Child costs and usage roll
up to the parent, and memory providers receive an `on_delegation` hook.

This design gives Hermes parallel reasoning and work decomposition while keeping
child context isolated. The child can read files, run tools, and produce a
summary without polluting the parent's main prompt with every intermediate
detail.

## Plugins And Extensibility

`hermes_cli/plugins.py` discovers plugins from bundled directories, user
plugins, project opt-in plugins, and Python entry points.

Plugins can register:

- tools
- lifecycle hooks
- CLI commands
- context engines
- provider backends
- platform adapters
- transformations for terminal commands, tool results, and LLM output

The plugin manager intentionally skips special plugin families such as memory
providers, context engines, and model providers in some discovery paths because
those systems have their own import lifecycles. This avoids double-registration
and keeps provider profiles lazy.

Most hooks are fail-open: a broken plugin logs and returns control to the main
agent rather than taking down the run. Blocking hooks are explicit, such as
`pre_tool_call` returning a block directive.

## Harness And Trajectories

Hermes also has a training/evaluation harness story.

`batch_runner.py` processes datasets in batches, samples toolsets from
distributions, runs `AIAgent` with context files and memory disabled, extracts
tool/reasoning stats, and writes normalized JSONL trajectory entries.

`AIAgent._convert_to_trajectory_format` turns the internal message list into a
training-style conversation format with:

- a synthetic system message containing tool signatures
- the original user prompt
- assistant turns with `<think>` blocks
- tool calls wrapped in `<tool_call>`
- tool results wrapped in `<tool_response>`

That means the same runtime used interactively can also generate reusable
agentic traces.

## Persistence Model

`hermes_state.py` stores sessions in SQLite. The schema records:

- session metadata
- source/platform
- model configuration
- system prompt snapshot
- parent session ID
- message content
- tool calls
- reasoning fields
- Codex Responses reasoning/message items
- token and cost counters

Design details worth noting:

- WAL mode is used for concurrent readers and one writer, with fallback to
  DELETE journal mode on filesystems where WAL locking fails.
- FTS5 indexes message content, tool names, and tool call data.
- A trigram FTS5 table supports substring/CJK search.
- Compression creates linked child sessions via `parent_session_id`.
- Resume and list operations project compression roots forward to their live
  tips so long-running compressed conversations still appear coherent.

This persistence layer is what makes `/resume`, session search, title/history,
compression lineage, and cross-platform gateway sessions feel unified.

## Why The Architecture Works

Hermes is powerful because it keeps several invariants:

- The internal message history is richer than any single provider wire format.
- Provider-specific formatting happens at the boundary, not throughout the loop.
- The system prompt is stable for caching; volatile context is injected through
  user-message additions, tools, prefetch, or compression boundaries.
- Tool schemas come from the registry/toolset/plugin system, not hardcoded
  prompt text.
- Large data is kept in files or auxiliary summaries, not blindly stuffed into
  context.
- Subagents are real agents with isolation, not mere prompts to the same model.
- Plugins extend generic surfaces rather than forcing plugin-specific branches
  into the core.
- Session persistence treats long conversations, compressed sessions, and child
  work as related records instead of throwaway chat logs.

The result is an agent runtime that can run locally, in messaging platforms, in
batch jobs, and inside delegated workers while preserving one core execution
model.

## Where To Start Reading

For a first code pass:

1. `run_agent.py`
   - constructor
   - `_build_system_prompt_parts`
   - `run_conversation`
   - `_execute_tool_calls`
   - `_compress_context`
2. `model_tools.py`, `tools/registry.py`, and `toolsets.py`
3. `agent/prompt_builder.py`
4. `agent/context_engine.py` and `agent/context_compressor.py`
5. `tools/memory_tool.py`, `agent/memory_provider.py`, and
   `agent/memory_manager.py`
6. `tools/delegate_tool.py`
7. `tools/code_execution_tool.py`
8. `hermes_state.py`
9. `hermes_cli/plugins.py`
10. `batch_runner.py`

The companion notes file gives a more detailed source map and implementation
trail.
