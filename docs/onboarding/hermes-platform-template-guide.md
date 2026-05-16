# Hermes Platform Template Guide

This guide captures the reusable architecture behind Hermes so it can be used
as a template for new agent platforms in other languages and for niche
applications built on top of the same ideas.

It is not a recommendation to copy Hermes line-for-line. Hermes is a large
general-purpose local agent runtime. A product like Meridian should preserve
the architectural invariants while shrinking, renaming, and specializing the
surfaces around its domain.

## Core Thesis

The reusable unit is not "a chatbot with tools." The reusable unit is an
agentic application platform:

```text
Domain UI / API / scheduler
        |
        v
Agent runtime kernel
  - provider adapters
  - prompt builder
  - tool/capability registry
  - context manager
  - memory and retrieval
  - orchestration queue
  - persistence/event log
  - review and safety gates
        |
        v
Domain state and deterministic effects
```

Hermes proves this shape in a broad developer-agent setting. A niche app should
make the same shape feel native to the user and the problem domain.

## Template Layers

### 1. Runtime Kernel

The runtime kernel owns one run from user/task input to final result. In Hermes
this is `AIAgent.run_conversation`; in a smaller app it might be
`AgentRuntime.run(job:)`.

Responsibilities:

- hold the current run state
- build provider-facing messages from internal state
- call the active provider
- parse and validate model output
- execute capabilities or route subjobs
- persist events and outputs
- enforce budgets, retries, and safety rules
- return a typed result

Keep the kernel boring and explicit. The kernel should know the lifecycle, but
domain-specific work should live in tools, queues, stores, or reducers.

### 2. Provider Adapter Boundary

Hermes keeps a rich internal message format and projects it into provider wire
formats at the edge. Preserve that pattern in every language.

Template interface:

```text
ProviderAdapter
  complete(request: ModelRequest) -> ModelResponse

ModelRequest
  role / task id
  model
  system prompt
  user/context prompt
  tools or structured-output schema
  temperature / token limits

ModelResponse
  visible text or structured JSON
  tool calls, if supported
  token usage
  provider/model identity
  raw provider metadata, if useful
```

Rules:

- Provider-specific quirks belong in adapters, not product logic.
- The internal event/message model can be richer than any provider schema.
- Preserve enough metadata to replay, debug, and compare providers.
- Normalize empty responses, malformed JSON, and provider errors into runtime
  errors the app can display or recover from.

### 3. Prompt Builder

Hermes separates prompt layers: stable identity, project context, volatile
memory/session metadata. A niche app should do the same, but in domain language.

Template prompt inputs:

- role identity
- mission and responsibilities
- allowed actions
- output schema
- safety and privacy rules
- domain workspace snapshot
- recent memory or history
- source excerpts or citations
- current trigger and goal

Rules:

- Keep stable role prompts stable across runs where possible.
- Put changing workspace state in a run-specific user/context prompt.
- Use typed output schemas when app state will be mutated.
- Treat prompts as code: version them, test them, and document the contract.

### 4. Capability Registry

Hermes has a tool registry and toolsets. In a niche app, tools may be ordinary
domain capabilities rather than shell/file/browser tools.

Template capability examples:

- read workspace slice
- summarize document
- refresh source
- extract opportunity
- rank candidate
- draft email
- create review item
- update local model through a reducer
- schedule follow-up

Rules:

- Capabilities should be named contracts with typed inputs and outputs.
- Side effects should be deterministic and auditable.
- Dangerous effects should create reviewable proposals, not execute directly.
- Capabilities can be grouped into role-specific toolsets.
- The model should only see capabilities relevant to the current role.

### 5. Context Manager

Hermes combines context files, subdirectory hints, large-result persistence,
compression, memory prefetch, and session search. A niche app needs the same
idea in domain form.

Template responsibilities:

- decide which workspace facts fit into the current prompt
- include source excerpts rather than entire documents
- keep large files in local storage and pass summaries or references
- summarize old run history when needed
- retrieve relevant prior decisions or memories
- protect recent tail context
- keep citations/source names attached to claims

Rules:

- Never blindly inject the whole workspace.
- Prefer compact, source-grounded snapshots.
- Separate current workspace state, durable memory, and historical transcripts.
- Make truncation visible enough that the agent knows when it lacks full data.

### 6. Memory And Retrieval

Hermes separates built-in memory, external memory providers, and session search.
That separation is highly reusable.

Template memory types:

- durable profile facts
- domain preferences
- role-specific lessons
- prior decisions
- historical run transcripts
- source snapshots and change history

Rules:

- Durable memory should contain stable facts, not task logs.
- Historical search should recall prior work without polluting every prompt.
- Role memory should be compact and reviewable.
- Memory writes should be explicit outputs or deterministic reducer decisions.
- Sensitive memory should stay local unless the user opts in.

### 7. Orchestration Queue

Hermes subagents are isolated child agents. A niche app can implement the same
pattern as a queue of role jobs.

Template job:

```text
AgentJob
  id
  role
  trigger
  goal
  input summary
  status
  attempt
  created/started/completed timestamps
```

Template orchestration:

1. Build jobs from the trigger and workspace state.
2. Run each job through the runtime with role-specific prompt and context.
3. Decode typed output.
4. Apply deterministic side effects or create review proposals.
5. Persist memory, task status, usage, and errors.
6. Decide whether follow-up jobs are needed.

Start sequential. Add parallelism only when roles are independent and state
reducers are conflict-safe.

### 8. Review Gate

Hermes uses approvals, guardrails, and tool blocking for dangerous operations.
In a product app, the central safety primitive should be a review queue.

Use review gates for:

- outreach messages
- application submissions
- profile inferences
- opportunity creation from weak evidence
- document-derived claims
- settings/privacy changes
- anything sent outside the local device

The model can propose. The app decides what can be applied immediately and what
needs a user-visible approval step.

### 9. Persistence And Event Log

Hermes uses SQLite session storage with messages, tool calls, prompts, usage,
and compression lineage. A niche app needs a domain event log even when it also
has a normal app state store.

Persist:

- runs and triggers
- prompts or prompt versions
- provider/model used
- inputs and compact workspace snapshots
- decoded outputs
- proposed changes
- applied deterministic changes
- errors and retries
- token/cost usage
- memory writes

This gives the app debuggability, provider comparisons, regression tests, and a
path toward replay.

### 10. Harness And Evaluation

Hermes uses batch runs, trajectories, session search, and tests. Niche apps
should build evaluation harnesses early.

Useful harnesses:

- sample-user end-to-end flow
- role prompt contract tests
- provider parity validation
- deterministic reducer tests
- malformed JSON recovery tests
- privacy/sensitive-data tests
- source-grounding tests
- snapshot tests for prompt shape

The goal is not to prove the model is always right. The goal is to prove the
platform handles model variability without corrupting app state or user trust.

## Language-Porting Blueprint

Use this mapping when building a Hermes-like platform in another language.

| Hermes concept | Portable abstraction | Notes |
| --- | --- | --- |
| `AIAgent` | `AgentRuntime` or `AgentKernel` | Owns one run/turn lifecycle. |
| `run_conversation` | `run(input)` / `run(job)` | Synchronous or async loop around model calls and capabilities. |
| Provider adapters | `ProviderAdapter` protocol | Keep provider quirks at the edge. |
| Internal messages | `AgentEvent` / `ConversationEvent` | Richer than provider wire formats. |
| Tool registry | `CapabilityRegistry` | Typed contracts, availability checks, groups/toolsets. |
| Toolsets | `CapabilitySet` / role permissions | Expose the smallest useful capability set. |
| System prompt builder | `PromptBuilder` | Stable role prompt plus run-specific context. |
| Memory store | `MemoryStore` / `ProfileMemory` | Durable facts and role lessons. |
| Session search | `RunHistorySearch` | Historical recall, not always-injected memory. |
| Context compressor | `ContextManager` / `Summarizer` | Budget-aware snapshots and history summaries. |
| Subagents | `AgentJobQueue` / `WorkerAgent` | Isolated role runs with typed handoff. |
| Tool guardrails | `SafetyPolicy` / `ReviewPolicy` | Warnings, blocks, approvals, review proposals. |
| `state.db` | `RunStore` / event log | Replay/debug/eval record. |
| Batch runner | `EvaluationHarness` | Sample flows and provider comparisons. |

Recommended minimal module layout:

```text
AgentRuntime/
  AgentRuntime
  AgentJob
  AgentEvent
  AgentRunResult
Providers/
  ProviderAdapter
  OpenAIAdapter
  AnthropicAdapter
Prompts/
  RolePromptDefinition
  PromptBuilder
Capabilities/
  Capability
  CapabilityRegistry
  CapabilitySet
Context/
  ContextSnapshot
  ContextManager
  HistorySummarizer
Memory/
  MemoryStore
  RunHistorySearch
Orchestration/
  AgentQueue
  Scheduler
  ReviewQueue
Persistence/
  RunStore
  WorkspaceStore
Evaluation/
  SampleFlowTests
  ProviderValidation
```

## Applying The Template To Meridian

Meridian already follows several Hermes-inspired patterns:

- native local-first app state
- role prompts for Orchestrator, Profile, Source Bootstrap, Source,
  Opportunity, Ranking, Action, and Settings & Privacy agents
- an offline agent queue
- provider clients for OpenRouter, OpenAI, and Anthropic
- typed JSON response contract
- local memory entries
- deterministic side effects after model output
- source/opportunity/action domain models
- sample-student and live-provider validation

The next step is not to make Meridian a general Hermes clone. The next step is
to deepen the platform pattern in Meridian-specific language.

### Meridian Runtime Mapping

| Hermes platform idea | Meridian-specific form |
| --- | --- |
| Agent runtime | `MeridianAgentOrchestrator` plus a future `MeridianAgentRuntime` extracted around one job run. |
| Tool/capability registry | Domain capabilities: refresh source, extract opportunity, rank, draft action, summarize document, create review item. |
| Toolsets | Role permissions: Source agents get source tools; Action gets draft/review tools; Settings gets provider/privacy checks. |
| Context files | Workspace snapshot: profile, watched sources, excerpts, opportunities, actions, documents, recent role memory. |
| Subagents | `OfflineAgentQueue` role jobs. |
| Memory | Student profile facts, role memories, source trust lessons, ranking preferences, outreach tone lessons. |
| Session search | Search prior agent runs, source snapshots, and review decisions. |
| Compression | Compact older run history and source excerpts into summaries with source IDs. |
| Guardrails | Review queues for outreach, submissions, sensitive docs, profile inferences, source promotion. |
| Harness | Sample student workflow, provider validation, source-grounding fixtures, privacy tests. |

### Meridian-Specific Architecture Target

```text
SwiftUI screens
  Today / Discover / Sources / Action Center / Profile / Settings
        |
        v
MeridianAgentRuntime
  role prompt + workspace context + provider adapter
        |
        v
Meridian capability layer
  source refresh
  source extraction
  opportunity normalization
  ranking
  draft/action creation
  profile inference
  privacy checks
        |
        v
Review queue + deterministic store reducers
        |
        v
Local workspace, run history, memory, source snapshots
```

### Meridian Next Captures

These are the Hermes lessons most worth capturing as Meridian evolves:

- Extract a small `MeridianAgentRuntime` around the per-job provider call,
  JSON decoding, error handling, memory write, deterministic side effects, and
  task status updates. Keep `MeridianAgentOrchestrator` focused on scheduling.
- Introduce a typed domain capability registry before adding live crawl/search.
  This keeps source refresh, document summarization, opportunity extraction,
  and action drafting testable.
- Add a run/event store separate from the user-facing workspace state. Store
  trigger, role, prompt version, compact context snapshot, provider/model,
  decoded output, proposed changes, applied changes, token usage, and errors.
- Treat source snapshots as first-class context artifacts. Store raw fetch
  result, extracted text, hash/change signal, summary, and opportunity
  extraction status separately.
- Add a review queue type for proposed changes. The current `proposedChanges`
  string array is a good start; long term, use typed proposals such as
  `ProfileInferenceProposal`, `OpportunityCandidate`, `ActionDraft`, and
  `SourceTrustUpdate`.
- Keep outreach and applications proposal-only until the student approves.
- Make prompt definitions versioned so old run outputs can be interpreted
  against the exact prompt contract that produced them.
- Add context budgets. Do not send every source, document, and opportunity once
  real user data grows.
- Add history search over prior agent runs and review decisions. This is the
  Meridian equivalent of Hermes session search.
- Keep API keys in Keychain and never include key values in prompts, logs,
  run history, exports, or validation artifacts.

### Meridian Capability Examples

Meridian capabilities should be domain-native:

```text
refreshSource(sourceID) -> SourceSnapshot
summarizeSource(snapshotID) -> SourceSummary
extractOpportunity(snapshotID) -> [OpportunityCandidate]
rankOpportunities(profileID, opportunityIDs) -> RankingProposal
draftOutreach(opportunityID, profileContextID) -> ActionDraft
inferProfileFacts(workspaceSliceID) -> [ProfileInferenceProposal]
checkPrivacy(workspaceSliceID) -> [PrivacyWarning]
createReviewItems(proposals) -> [ReviewItem]
```

The model can request or produce these operations, but the app should apply
them through deterministic reducers that can be tested without a model.

### Meridian Context Snapshot Shape

A useful prompt context should be compact and source-grounded:

```text
AgentContextSnapshot
  trigger
  role goal
  profile summary
  relevant source summaries and excerpts
  relevant opportunity cards
  due/open actions
  relevant documents metadata, not full sensitive content by default
  recent role memory
  prior review decisions
  explicit unknowns
```

Do not let a growing workspace become one giant prompt. Build context selection
as a product feature.

## Minimal Platform Build Order

For a new language or niche app, build in this order:

1. Define internal run/event models.
2. Add one provider adapter and one structured-output response path.
3. Add role prompt definitions and prompt contract tests.
4. Add a local workspace store and run history store.
5. Add one deterministic capability and one review proposal type.
6. Add the runtime loop around provider call, decode, reducer, persistence.
7. Add an orchestration queue for multiple role jobs.
8. Add memory as explicit compact facts.
9. Add context selection and budgets.
10. Add source/history retrieval.
11. Add provider parity validation.
12. Add subagent parallelism only after state conflicts are solved.

This order avoids the common trap of adding many agents before the runtime,
state, review, and evaluation foundations can support them.

## Patterns To Preserve

- Rich internal events, provider-specific projection at the boundary.
- Stable role prompts, changing workspace context outside the stable prompt.
- Typed model output for anything that mutates app state.
- Deterministic reducers for applying model output.
- Domain capabilities instead of arbitrary universal tools.
- Review gates for external side effects and uncertain inferences.
- Compact memory for stable facts; search for historical runs.
- Context budgets from the beginning.
- Provider parity tests and sample-user workflows.
- Local-first privacy posture when the app handles personal data.

## Anti-Patterns To Avoid

- Rebuilding a full general-purpose terminal/file agent inside a niche app.
- Letting the user see or manage internal agent routing.
- Letting provider-specific request logic leak into domain stores or views.
- Stuffing the whole local database into every prompt.
- Treating all model suggestions as applied facts.
- Saving task logs as durable memory.
- Using one giant "do everything" prompt after the domain has clear role
  boundaries.
- Adding parallel subagents before deterministic merge/review semantics exist.
- Logging or exporting sensitive provider keys or document content.

## Reusable Design Checklist

Before calling a new app "Hermes-inspired," verify it has answers for:

- What is the internal event/run model?
- What is the stable role prompt contract?
- What changing context enters each run?
- What provider adapters exist, and where are quirks isolated?
- What capabilities can the model use or propose?
- Which effects are immediate and which require review?
- Where do durable memories live?
- Where do historical transcripts or run outputs live?
- How is context kept under budget?
- How are model outputs validated, repaired, or rejected?
- How are failed runs retried or surfaced?
- How are provider/model differences tested?
- How can a developer replay or audit a run?

If these are explicit, the architecture can travel across languages and product
niches without losing the qualities that make Hermes effective.
