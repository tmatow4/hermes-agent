# Hermes Core Architecture Notes

These notes are a source map for the architecture report. They emphasize where
the important implementation decisions live and what to verify before changing
them.

## High-Level Source Map

| Area | Primary files | Notes |
| --- | --- | --- |
| Agent runtime | `run_agent.py` | `AIAgent`, provider resolution, prompt cache, loop, tool execution, compression, persistence. |
| Tool orchestration | `model_tools.py`, `tools/registry.py`, `toolsets.py` | Registry discovery, toolset exposure, schemas, dispatch, dynamic `execute_code` schema. |
| Prompts | `agent/prompt_builder.py`, `run_agent.py` | Identity, context files, skills prompt, model/platform guidance, memory injection. |
| Context compression | `agent/context_engine.py`, `agent/context_compressor.py` | Context engine interface and built-in compression implementation. |
| Subdirectory context | `agent/subdirectory_hints.py` | Lazy discovery of deeper `AGENTS.md`/`CLAUDE.md`/rules files from tool arguments. |
| Tool result budgets | `tools/tool_result_storage.py` | Persist oversized tool outputs to backend temp storage and keep previews in context. |
| Built-in memory | `tools/memory_tool.py` | Bounded file-backed `MEMORY.md` and `USER.md` snapshot. |
| External memory | `agent/memory_provider.py`, `agent/memory_manager.py` | Provider ABC, one-provider manager, prefetch/sync/tool/lifecycle hooks. |
| Session state | `hermes_state.py` | SQLite `state.db`, FTS search, compression lineage, message persistence. |
| Session recall | `tools/session_search_tool.py` | FTS search plus auxiliary-model summaries. |
| Delegation | `tools/delegate_tool.py` | Child `AIAgent` construction, restricted tools, parallel child execution. |
| Programmatic tool calling | `tools/code_execution_tool.py` | Python harness that calls Hermes tools over local or remote RPC. |
| Plugins | `hermes_cli/plugins.py` | Plugin discovery, registration context, hooks, context engines, tools. |
| Batch harness | `batch_runner.py` | Parallel dataset processing, tool stats, trajectory saving. |
| Provider adapters | `agent/codex_responses_adapter.py`, `agent/anthropic_adapter.py`, `agent/bedrock_adapter.py` | Wire-format projection and response normalization helpers. |
| Existing docs | `website/docs/developer-guide/*.md` | Useful overview docs, but verify current behavior in source before editing core. |

## Current Source Landmarks

These line ranges are from the codebase at the time this research branch was
created. Treat them as starting points, not permanent anchors.

| Topic | Landmark |
| --- | --- |
| Iteration budget | `run_agent.py:287-330` |
| Tool parallelization rules | `run_agent.py:332-452` |
| `AIAgent.__init__` signature | `run_agent.py:1136-1202` |
| API mode inference and Responses upgrade | `run_agent.py:1289-1371` |
| Tool schema loading | `run_agent.py:1863-1884` |
| Built-in memory setup | `run_agent.py:1993-2014` |
| External memory provider setup | `run_agent.py:2018-2102` |
| Context engine setup | `run_agent.py:2128-2409` |
| System prompt assembly | `run_agent.py:6056-6281` |
| Pre-call message sanitizer | `run_agent.py:6314-6382` |
| Thinking-only cleanup | `run_agent.py:6384-6505` |
| Compression coordinator | `run_agent.py:10656-10830` |
| Tool execution entry | `run_agent.py:10875-10897` |
| Concurrent tool invocation | `run_agent.py:10917-11003` |
| Sequential tool result append path | `run_agent.py:11780-11875` |
| Main conversation loop | `run_agent.py:12094-16028` |
| Tool-call branch and compression check | `run_agent.py:15038-15335` |
| Trajectory conversion | `run_agent.py:4803-5010` |
| Prompt builder identity/guidance | `agent/prompt_builder.py:134-409` |
| Context file loading | `agent/prompt_builder.py:1289-1456` |
| Environment hints | `agent/prompt_builder.py:592-820` |
| Registry discovery/registration/dispatch | `tools/registry.py:57-416` |
| Tool definition cache/schema compute | `model_tools.py:242-474` |
| Normal tool dispatch | `model_tools.py:731-870` |
| Core tool list and toolsets | `toolsets.py:29-260` |
| Toolset resolution | `toolsets.py:590-752` |
| Context engine ABC | `agent/context_engine.py:1-101` |
| Compressor state and thresholds | `agent/context_compressor.py:420-513` |
| Compressor summary generation | `agent/context_compressor.py:793-964` |
| Compressor main `compress` flow | `agent/context_compressor.py:1374-1583` |
| Built-in memory store | `tools/memory_tool.py:1-500` |
| Memory provider ABC | `agent/memory_provider.py:1-31` |
| Memory manager | `agent/memory_manager.py:1-455` |
| Session DB schema and FTS | `hermes_state.py:1-315` |
| Session DB compression lineage | `hermes_state.py:1220-1350` |
| Message persistence | `hermes_state.py:1550-1619` |
| Session search | `tools/session_search_tool.py:1-520` |
| Delegate tool child construction | `tools/delegate_tool.py:865-1169` |
| Delegate execution | `tools/delegate_tool.py:1316-2304` |
| Execute-code architecture | `tools/code_execution_tool.py:1-29` |
| Execute-code remote env/RPC | `tools/code_execution_tool.py:563-940` |
| Execute-code local child env | `tools/code_execution_tool.py:1140-1230` |
| Plugin registration surface | `hermes_cli/plugins.py:128-350` |
| Plugin loading and hook invoke | `hermes_cli/plugins.py:741-1435` |
| Batch runner agent instantiation | `batch_runner.py:323-379` |

## Runtime Skeleton

`AIAgent.run_conversation` is the main lifecycle:

```text
ensure session
append user message
build or reuse system prompt
preflight compression
plugin pre_llm_call context injection
while budget remains:
    prepare provider-facing message copy
    inject memory prefetch and plugin context into latest user message
    call model transport
    normalize response
    if response has tool calls:
        validate/repair tool calls
        append assistant tool-call message
        execute tools
        append tool result messages
        maybe compress
        continue
    else:
        finalize response
        persist, sync, hook, return
if exhausted:
    request toolless summary
```

Important implementation checkpoints:

- `IterationBudget` tracks max tool-calling iterations and supports refunds.
- `run_conversation` resets iteration budget per user turn.
- Tool-only loops are bounded by both iteration budget and tool loop guardrails.
- `execute_code`-only turns refund an iteration after tool execution.
- The stored message history is richer than the provider-facing copy.

## AIAgent Initialization Notes

Important constructor behavior in `run_agent.py`:

- API mode can be explicit or inferred from provider/base URL.
- GPT/Codex-style direct OpenAI requests are upgraded to `codex_responses` when
  appropriate.
- Callback fields cover streaming, reasoning, tool progress, status, approvals,
  interrupts, and gateway/TUI integrations.
- Prompt caching settings are established before the first call.
- Tool definitions are loaded once per agent instance, but tool schema caches
  in `model_tools.py` are invalidated by registry/config generation changes.
- Built-in memory and external memory provider setup are independent.
- Context engine selection prefers configured plugins when not using the
  built-in compressor.

Risk when changing this area: constructor state is shared by many entry points,
including gateway, TUI, batch, subagents, and tests. Small default changes can
surface in surprising channels.

## Prompt Assembly Notes

Prompt assembly has two code locations:

- `agent/prompt_builder.py` provides prompt fragments and context file loading.
- `AIAgent._build_system_prompt_parts` orders runtime-specific pieces.

Stable tier includes:

- `SOUL.md` or default identity
- Hermes help guidance
- memory/session/skills/kanban/computer-use guidance gated by available tools
- Nous subscription guidance when relevant
- tool-use enforcement guidance
- model-family guidance for GPT/Codex/Gemini/Gemma
- skills prompt
- provider identity workaround for Alibaba
- environment hints
- platform hints

Context tier includes:

- caller-provided system message
- project context files

Volatile tier includes:

- built-in memory snapshot
- external memory provider system prompt block
- conversation start timestamp
- optional session ID, model, provider

Important nuance:

- Comments describe volatile content as per-session/turn, but
  `_build_system_prompt` joins every tier into one cached string.
- On resume, the exact stored system prompt can be reused from `state.db`.
- Mid-session memory writes do not update the injected memory prompt until the
  prompt is rebuilt, such as after compression or in a new session.
- `ephemeral_system_prompt` is intentionally not part of the stored/cached
  system prompt. It is injected at API-call time.

## Context File Loading

`build_context_files_prompt` loads at most one project context type, in priority
order:

1. `.hermes.md` or `HERMES.md`, walking up to git root.
2. `AGENTS.md` or `agents.md`, cwd only.
3. `CLAUDE.md` or `claude.md`, cwd only.
4. `.cursorrules` or `.cursor/rules/*.mdc`, cwd only.

Context files are scanned for obvious prompt-injection patterns and invisible
characters before injection. Large files are truncated with head/tail retention
and a marker telling the agent to use file tools for the full file.

Gateway mode uses `TERMINAL_CWD` for context file discovery so the gateway
process does not accidentally load the Hermes repo's own context files.

## Tool Registry And Toolsets

`tools/registry.py`:

- Scans `tools/*.py` for top-level `registry.register(...)`.
- Imports discovered files to populate the registry.
- Stores entries, aliases, check functions, dynamic schemas, and max result
  sizes.
- Increments a generation counter on registration so schema caches can
  invalidate.
- Dispatches sync or async handlers and returns JSON strings.

`toolsets.py`:

- Defines `_HERMES_CORE_TOOLS`.
- Defines named toolsets such as file, terminal, memory, delegation, browser,
  web, skills, messaging, kanban, and computer-use.
- Resolves `all`, recursive inheritance, plugin toolsets, and
  platform-specific generated toolsets.

`model_tools.py`:

- Discovers built-in tools and plugins.
- Resolves enabled/disabled toolsets.
- Applies availability checks.
- Rebuilds the `execute_code` schema based on which sandbox-callable tools are
  currently enabled.
- Sanitizes tool schemas.
- Routes normal dispatch through `handle_function_call`.

Agent-loop-only tools:

- `todo`
- `memory`
- `session_search`
- `delegate_task`

These are intercepted in `run_agent.py` because they require the live agent,
session, memory manager, or parent-agent context.

## Tool Execution Notes

The main execution boundary is `_execute_tool_calls`, which chooses sequential
or parallel execution.

Parallelization is allowed only when:

- no tool is in the never-parallel set, such as `clarify`
- arguments parse as dictionaries
- path-scoped tool calls do not overlap
- non-read-only tools are explicitly safe or come from an MCP server that opts
  into parallel calls

Both sequential and parallel paths:

- run pre-tool plugin hooks
- handle agent-level tools
- route memory-provider tools
- route context-engine tools
- call the registry for ordinary tools
- persist oversized results
- append subdirectory hints
- emit callbacks
- record guardrail observations
- append `role=tool` messages
- enforce aggregate tool-result budgets

Tool loop guardrails live in `agent/tool_guardrails.py`. They detect repeated
exact failures, repeated same-tool failures, and repeated read-only calls that
return identical results. Warnings append guidance to tool results; hard stops
are optional config and create a controlled halt response.

## Large Result Strategy

`tools/tool_result_storage.py` defines three layers:

1. Individual tools may truncate their own output.
2. `maybe_persist_tool_result` writes an oversized single result to the active
   sandbox/backend temp directory and returns a preview plus path.
3. `enforce_turn_budget` spills the largest non-persisted results when a whole
   turn exceeds the aggregate budget.

The storage path is resolved from the active environment when possible, so a
model working in Docker/SSH/Modal/etc. gets a path it can read from the same
backend.

This is a major context-safety mechanism. It preserves access to full data
without forcing full data into the LLM prompt.

## Context Compressor Notes

The context engine ABC is in `agent/context_engine.py`. The built-in compressor
is `ContextCompressor`.

Default compressor behavior:

- Tracks last prompt token count and context length.
- Applies configured threshold/target behavior, with model-specific overrides.
- Avoids compression thrash when recent compressions save too little.
- Prunes old tool results before summarizing.
- Protects a recent tail window.
- Summarizes middle history into a structured "reference only" summary.
- Preserves assistant/tool result pairing.
- Sanitizes orphan tool pairs after compression.
- Inserts fallback markers if summary generation fails.

`AIAgent._compress_context` coordinates with persistence and memory:

- calls external memory `on_pre_compress`
- calls the context engine compressor
- appends a current todo snapshot when available
- invalidates and rebuilds the system prompt
- ends the old session with reason `compression`
- creates/uses a child session linked with `parent_session_id`
- tells context engine and memory provider about the session switch
- resets file-read dedupe state

Observation to verify before editing:

- The memory manager's `on_pre_compress` hook is invoked before compression. In
  the current code pass, do not assume the returned text is automatically folded
  into the compressor prompt without tracing the exact call path.

## Memory Notes

Built-in memory (`tools/memory_tool.py`):

- Stores durable facts in `MEMORY.md` and user facts in `USER.md`.
- Has count/character limits.
- Scans writes for injection and exfiltration patterns.
- Uses file locks and atomic replacement.
- Captures a `_system_prompt_snapshot` when loaded.
- Formats prompt blocks from that frozen snapshot.
- Handles only `add`, `replace`, and `remove`.

External memory (`agent/memory_provider.py`, `agent/memory_manager.py`):

- Allows one active external provider.
- Provider interface includes `initialize`, `system_prompt_block`, `prefetch`,
  `sync_turn`, `shutdown`, optional tools, and lifecycle hooks.
- `MemoryManager` fences prefetched context in a memory-specific block and can
  scrub leaked context tags from output streams.
- Provider tools are indexed by name and routed before normal registry
  dispatch.
- Memory sync happens after final response handling.

Session recall (`tools/session_search_tool.py`):

- Searches SQLite FTS5.
- Groups results by resolved parent/root session.
- Excludes the current conversation lineage.
- Loads transcript windows around matches.
- Summarizes sessions in parallel through the auxiliary model router.

## SessionDB Notes

`hermes_state.py` is the durable conversation store.

Key design details:

- SQLite schema version is declarative for columns, with version-gated data
  migrations where necessary.
- WAL mode is preferred for concurrency; DELETE journal fallback supports
  network/FUSE filesystems.
- Message rows persist tool calls, tool names, reasoning fields, and Codex
  Responses items.
- FTS5 indexes content, tool names, and tool call JSON.
- Trigram FTS supports substring and CJK search.
- Compression creates parent/child session chains.
- Listing sessions can project compression roots to their live tips.
- Resume can redirect an empty compression parent to the descendant that holds
  messages.

Important distinction:

- Batch runner and RL trajectories are intentionally not stored in `state.db`.
  They are separate harness outputs.

## Subagent Notes

`tools/delegate_tool.py` implements delegation.

Important mechanics:

- Child agents are built on the main thread before worker execution to avoid
  global toolset contamination.
- Child tools are inherited from the parent and then restricted.
- `delegate_task`, `clarify`, `memory`, `send_message`, and `execute_code` are
  blocked for normal child agents.
- `role="orchestrator"` can retain delegation for controlled nested work.
- Child prompt includes goal, context, workspace rules, and summary
  expectations.
- Child `AIAgent` is constructed with quiet mode, skipped context files, skipped
  memory, parent session ID, and an ephemeral child prompt.
- Single child runs directly; batch children run in a thread pool.
- Parent interrupts propagate to active children.
- Approval callback defaults to non-interactive safety behavior and auto-denies
  dangerous commands unless config allows auto-approval.
- Child usage/cost rolls up to the parent.
- Memory providers receive an `on_delegation` hook.

Design implication:

- Subagents are isolated workers, not additional messages in the parent's
  context. The parent gets summaries and metadata, which keeps the parent prompt
  smaller and easier to steer.

## Programmatic Tool Calling Notes

`tools/code_execution_tool.py` lets model-written Python call Hermes tools
through a generated `hermes_tools.py` module.

Allowed sandbox tools are the intersection of session tools and:

- `web_search`
- `web_extract`
- `read_file`
- `write_file`
- `search_files`
- `patch`
- `terminal`

Local flow:

1. Parent generates the stub module.
2. Parent starts a socket RPC listener.
3. Child Python process runs the script.
4. Tool calls travel back to parent for dispatch.

Remote flow:

1. Parent creates or reuses the active terminal environment for the task.
2. Parent ships the script and stub module to the remote sandbox.
3. Script writes request files.
4. Parent polling thread dispatches tool calls and writes response files.
5. Script continues and prints final stdout.

Security/context controls:

- Child environment is scrubbed of API keys and secrets.
- Only configured env passthrough is allowed.
- Tool call count and timeout are capped.
- Only stdout is returned to the model.
- Intermediate tool results do not enter the model context.

This is an important performance and context feature because it lets a single
model tool call perform many deterministic file/search/tool operations.

## Plugin Notes

`hermes_cli/plugins.py` provides the generic plugin surface.

Discovery sources:

- bundled plugins
- user plugins under Hermes home
- project opt-in plugins
- pip entry points

Plugin context can register:

- tools
- slash/CLI commands
- hooks
- context engines
- provider backends
- platform backends

Notable hook families:

- pre/post tool call
- transform terminal command
- transform tool result
- pre/post LLM call
- pre/post API request
- transform LLM output
- session start/end
- subagent stop
- gateway dispatch
- approvals

Most hook failures are logged and ignored. Blocking behavior is explicit,
especially for `pre_tool_call`.

Design note:

- Memory providers, context engines, and model providers have specialized
  discovery/import paths. Avoid forcing plugin-specific imports into the main
  plugin manager unless the generic surface truly needs expansion.

## Provider Adapter Notes

`agent/codex_responses_adapter.py` is the clearest example of provider edge
normalization:

- Converts chat-style content to Responses input parts.
- Converts assistant tool calls to `function_call` items.
- Derives deterministic call IDs to preserve cache stability.
- Replays encrypted reasoning items when supported.
- Replays exact assistant message items with IDs/phase when available.
- Preflights `model`, `instructions`, `input`, `tools`, `store=false`, and
  allowed keys.

`run_agent.py` keeps provider-specific compatibility helpers near the main loop:

- drop thinking-only assistant turns from the provider-facing copy
- copy/pad reasoning content for providers that require it
- strip Responses-specific tool-call fields for strict chat-completions APIs
- repair corrupted tool-call argument JSON
- sanitize orphaned tool-call/tool-result pairs

The pattern is: keep internal history rich, then project and sanitize for the
active provider immediately before the API call.

## Harness Notes

`batch_runner.py`:

- Loads a dataset.
- Splits prompts into batches.
- Uses multiprocessing.
- Samples toolsets from a distribution.
- Instantiates `AIAgent` with `skip_context_files=True` and `skip_memory=True`.
- Runs each prompt with an isolated task ID.
- Extracts tool stats and reasoning stats.
- Converts messages to trajectory format through the agent method.
- Writes JSONL outputs with normalized stats.

`AIAgent._convert_to_trajectory_format`:

- Creates a synthetic tool-use system prompt.
- Emits the original user prompt.
- Emits assistant turns as `from: gpt`.
- Preserves reasoning in `<think>` blocks.
- Emits tool calls in `<tool_call>` blocks.
- Emits tool responses in `<tool_response>` blocks.

This keeps interactive runtime behavior and training/evaluation trace generation
closely aligned.

## Architectural Invariants

Useful rules to preserve when changing core code:

- Keep provider-specific wire-format logic at transport/adapter boundaries.
- Do not mutate stored history just to satisfy one provider's request schema;
  sanitize a per-call copy instead.
- Keep the system prompt stable across a session unless there is an intentional
  boundary such as compression.
- Put rapidly changing recall into user-message context, tool results,
  prefetch, or summaries rather than rebuilding the system prompt every turn.
- Prefer registering tools through the registry/plugin surfaces over editing
  the core dispatcher.
- Keep agent-level tools intercepted where live `AIAgent` state is available.
- Treat large outputs as files plus previews, not as prompt text.
- Keep subagent intermediate details out of the parent context unless the parent
  explicitly asks for them.
- Store durable user facts in memory, prior task history in session search, and
  current work state in the active conversation or todo tool.

## Good Follow-Up Reading Order

For someone onboarding into core architecture work:

1. `website/docs/developer-guide/architecture.md`
2. `website/docs/developer-guide/agent-loop.md`
3. `run_agent.py` around `AIAgent.__init__`
4. `run_agent.py` around `_build_system_prompt_parts`
5. `run_agent.py` around `run_conversation`
6. `model_tools.py`
7. `tools/registry.py`
8. `toolsets.py`
9. `agent/prompt_builder.py`
10. `agent/context_engine.py`
11. `agent/context_compressor.py`
12. `tools/memory_tool.py`
13. `agent/memory_provider.py`
14. `agent/memory_manager.py`
15. `tools/delegate_tool.py`
16. `tools/code_execution_tool.py`
17. `hermes_state.py`
18. `tools/session_search_tool.py`
19. `hermes_cli/plugins.py`
20. `batch_runner.py`

## Questions To Recheck Before Major Refactors

- Should the "volatile" system prompt tier stay cached exactly as it is, or
  should any volatile content move to per-call user-message injection?
- Should external memory `on_pre_compress` output become part of the compression
  prompt, if it is not already used by a selected context engine?
- Which pieces of `AIAgent` are constructor state versus per-turn state, and
  which entry points rely on that distinction?
- Are there any plugin hooks that rely on current fail-open behavior?
- Do subagent child sessions need different persistence or search visibility for
  a given feature?
- Does a provider-specific fix belong in the adapter/transport boundary instead
  of the main loop?
