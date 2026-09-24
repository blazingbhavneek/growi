# Local-LLM readiness plan

**OpenAI-compatible endpoints · enforced chat research · Elasticsearch semantic retrieval**

- Status: **plan only**. No product code has been changed. Nothing was run except
  reading code, the lockfile, and the published npm tarballs listed in §2.1.
- Written: 2026-09-24, against branch `feature/local-llm` (clean, HEAD `8ff9e7bf8d`).
- Audience: an implementer who has **not** read this code before. Every task
  names the files, the change, the reason, and the tests that prove it.

---

## 0. How to use this document

### 0.1 Reading order

1. §1 **Decisions at a glance**: what gets built by default, and what is optional.
2. §2 **Verified ground truth**: facts about the installed SDKs and the codebase.
   Read this before touching anything. Several "obvious" approaches are wrong
   for reasons listed here.
3. §3 **Process**: which kiro specs to open. One of them has to be an *amend
   spec* under `.claude/rules/spec-lifecycle.md`.
4. §4 **Phase 0 spikes**: short, throwaway experiments that confirm the risky
   assumptions before product code is written.
5. §5 / §6 / §7: the three objectives, each with analysis → design → tasks →
   tests → manual verification → optional larger scope.
6. §8 **Evaluation harness**: how to measure whether local models actually do
   more research and stop exiting early.
7. §9 **Cross-cutting**: config keys, i18n, security checklist, risk register,
   open decisions, command cheat sheet.

### 0.2 Phase order and dependencies

```
Phase 0  Spikes (no product code)                        ~1-2 days
   │
   ├── Phase 1  Objective 1: OpenAI-compatible provider   (independent)
   │
   ├── Phase 2  Objective 2: enforced research policy      (independent of Phase 1 in code,
   │                                                         but measured against local models
   │                                                         through Phase 1, or the Phase-0 stopgap)
   │
   └── Phase 3  Objective 3: semantic / hybrid retrieval   (reuses Phase 1's endpoint config
                                                            for the embedding model; plugs into
                                                            Phase 2's search wrapper as a mode)
```

Phases 1 and 2 can run in parallel on separate branches. Phase 3 should start after Phase 1
is merged, because its embedding model reuses the provider config added there.

### 0.3 Repository rules the implementer must follow

These are loaded automatically for Claude Code sessions, but a human or a weaker agent must
read them explicitly:

| Rule | Where | What it means for this work |
|---|---|---|
| Coding style | `.claude/rules/coding-style.md` | Named exports; pure functions extracted from framework wrappers (Mastra tools, Express handlers, React hooks); executors take their work-set as input; no `console.log`; comments only for non-obvious WHY; ≤ 800 lines per file; minimal barrel surfaces. |
| Testing | `.claude/rules/testing.md` | `pnpm vitest run <partial-name>` from `apps/app`, **never `npx`**. Use `mock<T>()` from `vitest-mock-extended` instead of `as any` / `as unknown as T`. Read the `essential-test-design` and `essential-test-patterns` skills before writing tests. |
| Import convention | `apps/app/.claude/rules/import-convention.md` | No file extensions in import specifiers inside `apps/app`. |
| MongoDB regex | `.claude/rules/mongodb-regex.md` | Any regex sent to Mongo uses `escapeStringForMongoRegex`, never `RegExp.escape`. |
| Security | `.claude/rules/security.md` | No secret in logs, errors, or API responses. Validate every client-supplied field. |
| Devcontainer | `.claude/rules/devcontainer.md` | `mongo:27017` and `elasticsearch:9200` are always reachable; do not ping them. Never run `pnpm install` concurrently with builds/tests. |
| Model rule | `.claude/rules/model.md` (path-scoped to `apps/app/src/server/models/**`) | New persisted collections: follow the Mongoose→Prisma migration rules (see §7.5 T3.2). |
| Activity recording | `apps/app/.claude/rules/activity-recording.md` | Every new **non-GET** route (`test-model`, `test-embedding`, `embeddings/backfill`) uses `addActivity`, and emits its activity **before** `res.apiv3()`. Add `SupportedAction` constants next to `ACTION_ADMIN_AI_SETTING_UPDATE` in `apps/app/src/interfaces/activity.ts` (both the constant and the action lists it is registered in). |
| Server boot imports | `apps/app/.claude/rules/server-boot-imports.md` | `@ai-sdk/*` (incl. embedding resolvers) and any new SDK load via dynamic `import()` only; the embedding sync/backfill modules must not pull provider SDKs into the boot graph. |
| Mongo in tests | `apps/app/.claude/rules/mongo-test-setup.md` | Integration tests that need Mongo (`page-embedding.integ.ts`, `aggregate-to-index.integ.ts`) use the shared setup and check `MONGO_URI` first; never spawn an embedded server unconditionally. |
| Package dependencies | `apps/app/.claude/rules/package-dependencies.md` | Read it before adding **any** dependency (only relevant if D2's optional `@ai-sdk/openai-compatible` is adopted). |
| kiro-impl orchestration | `.claude/rules/kiro-impl-orchestration.md` | If this is executed via `/kiro-impl`, print the orchestration plan block first and pass models explicitly. |

### 0.4 Command cheat sheet (for later, when implementing; this plan did not run them)

```bash
# from apps/app
pnpm vitest run ai-provider.spec
pnpm vitest run provider-availability-rule.spec
pnpm vitest run put-ai-settings.spec
pnpm vitest run research-policy.spec
pnpm vitest run elasticsearch.spec
pnpm vitest run aggregate-to-index.integ          # needs Mongo (always up in devcontainer)
VITE_ELASTICSEARCH_URI=http://elasticsearch:9200 pnpm vitest run elasticsearch.integ

# from repo root (before committing)
turbo run lint --filter @growi/app
turbo run test --filter @growi/app
turbo run build --filter @growi/app
```

---

## 1. Decisions at a glance

Each row is the **default this plan implements**. Alternatives and the reasons they were not
chosen are in the section referenced. §9.5 lists the few decisions an owner may want to
override. None of them blocks starting Phase 0 or Phase 1.

| # | Topic | Default (smallest viable scope) | Optional larger scope | § |
|---|---|---|---|---|
| D1 | Custom endpoint cardinality | **One** new fixed slot `openai-compatible` alongside the existing four providers | Multiple named endpoints (dynamic provider records) | 5.2, 5.9 |
| D2 | SDK used for the custom endpoint | Existing `@ai-sdk/openai` via `createOpenAI({ baseURL, apiKey }).chat(modelId)`, **no new dependency** | `@ai-sdk/openai-compatible` (parses `reasoning_content`, sends no `Authorization` header when keyless) | 5.3 |
| D3 | Keyless endpoints (local Ollama) | API key optional for this provider; the resolver always passes an **explicit placeholder** key (see the security note in §2.2) | Per-endpoint "requires key" flag | 5.4 |
| D4 | Tool-calling guarantee for custom models | Admin **attestation** checkbox on the provider (required for availability), plus an admin-only **"Test model"** probe that uses the *saved* config | Probe results persisted as a verification record keyed by (baseURL, modelId) and required before a model can be selected | 5.6 |
| D5 | Research enforcement mechanism | Code-side policy: (a) server-side **seed retrieval** before the loop, (b) chat-only **budgeted wrapper tools** that track novelty and return guidance, (c) per-step `prepareStep` that **removes tools** once research is done or the step budget is nearly spent, (d) an **empty-final-answer guard**, (e) `toolChoice: 'required'` on the first step only when seed hits exist (provider-dependent) | `isTaskComplete` scorer gate; a two-phase "research then answer" run | 6.4 |
| D6 | Where research policy lives | Chat only (`post-message.ts` + chat-only wrapper tools). `suggestPathAgent` untouched | Share the budget primitives with suggest-path later | 6.3 |
| D7 | Embeddings source | **App-side** embeddings through the same provider layer (OpenAI / OpenAI-compatible such as Ollama `/v1/embeddings`), stored as `dense_vector` | ES-managed `semantic_text` + inference endpoint (ES 9 only) | 7.3 |
| D8 | Vector persistence | New Mongo collection keyed by (page, embeddingVersion) and `$lookup`-ed by `aggregate-to-index.ts` into `revisionBodyEmbedded` (the field that already exists but is never populated) | Store on the revision document | 7.4 |
| D9 | Hybrid ranking | **App-side RRF** over two ES requests (existing lexical query + kNN), both carrying the identical permission filter | ES `retriever.rrf` / `linear` (version-dependent, forbids top-level `sort`) | 7.6 |
| D10 | Rollout surface | Hybrid retrieval **for the chat tool only**, behind config; global search UI unchanged | Hybrid mode in the main search page | 7.10 |
| D11 | Granularity | One vector per page (path + headings + first N chars of body) | Chunk-level nested vectors with `inner_hits` returning a line offset for `getPageContent` | 7.10 |

---

## 2. Verified ground truth

This is the evidence base. Where a claim was **not** verified, it is marked *(verify)*.

### 2.1 Installed versions (from `pnpm-lock.yaml`)

`node_modules` is **not installed** in this checkout. SDK behavior below was read from the
published tarballs of the exact locked versions (downloaded into a scratch directory, not the
repo).

| Package | `apps/app/package.json` range | Locked |
|---|---|---|
| `@mastra/core` | `^1.32.1` | **1.41.0** |
| `@mastra/ai-sdk` | `^1.4.1` | **1.4.4** |
| `@mastra/memory` | `^1.17.5` | 1.20.2 |
| `ai` | `^6.0.176` | **6.0.197** |
| `@ai-sdk/openai` | `^3.0.69` | **3.0.69** |
| `@ai-sdk/openai-compatible` | not a dependency | not in lockfile |
| `@elastic/elasticsearch8` (alias) | `npm:@elastic/elasticsearch@^8.18.2` | 8.19.0 |
| `@elastic/elasticsearch9` (alias) | `npm:@elastic/elasticsearch@^9.0.3` | 9.1.0 |
| Devcontainer ES image | `.devcontainer/compose.yml:80` | `elasticsearch:9.3.3` |

### 2.2 `@ai-sdk/openai` 3.0.69 facts (read from `dist/index.mjs`)

1. **`createOpenAI(opts)(modelId)` returns the Responses API model**, not Chat Completions.
   `provider(modelId)` → `createLanguageModel` → `createResponsesModel`. Chat Completions is
   `provider.chat(modelId)`. Most OpenAI-compatible servers (Ollama, vLLM, llama.cpp
   `llama-server`, LM Studio) implement `/v1/chat/completions` fully and `/v1/responses` only
   partially or not at all. **The custom-endpoint resolver must call `.chat(modelId)`.**
2. **API key fallback to the process environment.** `getHeaders()` calls
   `loadApiKey({ apiKey: options.apiKey, environmentVariableName: 'OPENAI_API_KEY' })`. If
   `apiKey` is `undefined` it reads `process.env.OPENAI_API_KEY`, and throws if that is
   also unset. **Security consequence:** a keyless custom endpoint that passed
   `apiKey: undefined` would send the operator's real `OPENAI_API_KEY` (if set in the
   environment) as a Bearer token to an arbitrary URL. **Always pass an explicit
   non-secret placeholder string.**
3. **Base URL fallback to the process environment.** `baseURL` is resolved with
   `loadOptionalSetting({ settingValue: options.baseURL, environmentVariableName: 'OPENAI_BASE_URL' })`
   and defaults to `https://api.openai.com/v1`. The custom resolver must never call
   `createOpenAI` with a blank `baseURL`. It throws before the call instead.
   - Side finding: the **existing** `openai` resolver
     (`apps/app/src/features/mastra/server/services/ai-sdk-modules/llm-providers/openai.ts:20-21`)
     passes no `baseURL`, so today an operator's `OPENAI_BASE_URL` env var silently redirects
     the `openai` provider. That contradicts the file's own comment ("explicit apiKey
     injection only (never the provider's process.env auto-detection)"). See §9.5 OD-6.
4. `withoutTrailingSlash` is applied to `baseURL`, and paths are appended (`/chat/completions`,
   `/embeddings`). So the admin enters `http://host:11434/v1`, **including** `/v1`.
5. The chat model parses provider options under the **hard-coded namespace `openai`**
   (`parseProviderOptions({ provider: "openai", ... })`), whatever `name` is passed to
   `createOpenAI`. So `providerOptions` for the new provider must be written as
   `{ "openai": { ... } }`, the same as `azure-openai` today
   (`client/admin/provider-options-namespace.ts:16-21`).
6. Useful escape hatches under that namespace for non-OpenAI models:
   `systemMessageMode: 'system' | 'developer' | 'remove'` and `forceReasoning: boolean`
   (the SDK decides "reasoning model" from the model-id prefix; a custom id such as
   `qwen3:14b` is treated as non-reasoning, which is the right default).
7. `prepareChatTools` turns an empty tool list into "no tools sent" (`tools.length ? tools : undefined`).
8. The chat model does **not** parse `reasoning_content` / `reasoning` fields. Thinking text
   from Ollama/vLLM reasoning models is dropped, and the final `content` still arrives. The
   `@ai-sdk/openai-compatible` 2.x package does parse them (both field names). That is the only
   functional reason to add that dependency later (D2 optional).

### 2.3 `@mastra/core` 1.41.0 agent-loop facts (read from `dist/agent/agent.types.d.ts`, `dist/loop/types.d.ts`, `dist/processors/index.d.ts`, and the agentic-loop implementation chunk)

Per-call options accepted by `agent.stream()` / `agent.generate()` that matter here:

| Option | Semantics (verified in implementation) | Use in this plan |
|---|---|---|
| `maxSteps` | Upper bound on LLM iterations. Does **not** force any tool use. | Derived from research budgets. |
| `stopWhen` | Evaluated only after a step that would otherwise continue (`isContinued`). Stopping there ends the run **after a tool step, with no final answer**. | **Do not use** to end research. Use `toolChoice: 'none'` instead. |
| `prepareStep(args)` | Runs before **every** step as an input-step processor. `args` has `stepNumber` (0-based), `steps` (previous `StepResult`s with tool calls/results), `messageList`, `requestContext`, `systemMessages`, `tools`, `toolChoice`, `activeTools`. It may return `toolChoice`, `activeTools`, `systemMessages` (replaces untagged system messages), `messages`, `modelSettings`, `providerOptions`, `model`. | Main enforcement hook. |
| `toolChoice` | `'auto' \| 'none' \| 'required' \| { type:'tool', toolName }`. **Mastra itself strips all tools from the request when `toolChoice === 'none'`** (`prepareToolsAndToolChoice` returns `tools: undefined`). That is provider-independent, because the model never sees the tools. `'required'` is forwarded to the provider and only works if the provider honors it. | `'none'` = "answer now", reliable. `'required'` = best-effort. |
| `activeTools` | Filters the tool map by **registration key**. An empty array sends no tools. | Can restrict step 0 to search only, etc. |
| `onIterationComplete(ctx)` | Runs after each LLM iteration with `{ iteration, isFinal, text, toolCalls, toolResults, finishReason, messages }`. Return `{ continue, feedback }`. **Trap:** `feedback` is only appended when the step was *already continuing*. On a final step (the model stopped), `{ continue: true, feedback }` reopens the loop **without** the feedback message. The feedback is also added as an **assistant**-role message. | Only for the empty-answer guard (nothing visible was streamed yet). The nudge is delivered through `prepareStep`, not `feedback`. |
| `isTaskComplete: { scorers, strategy }` | After a *final* step, runs scorers. On failure it appends an assistant-role feedback message, flips `isContinued = true`, and enqueues an `is-task-complete` chunk. | Optional strategy only (S6 in §8). Risks: a premature answer is already streamed; the feedback message may be persisted to memory; `@mastra/ai-sdk` 1.4.4 has **no** handler for `is-task-complete` (grep found none), so UI behavior must be spiked. |
| `context: ModelMessage[]` | Added to the message list with source `"context"`. | Delivery channel for seed retrieval. *(verify in Spike S0.3 that `context` messages are not persisted to the memory thread)*. |
| `onStepFinish`, `onFinish` | Observability callbacks. | Research trace logging. |

Other verified facts:

- `MastraModelConfig` accepts an `OpenAICompatibleConfig` (`{ providerId, modelId, url, apiKey, headers }`).
  That routes through Mastra's **model router / gateway** machinery. The completed spec
  `ai-provider-model-picker` explicitly rejected the Mastra model router (runtime
  models.dev fetch, fidelity drift; `.kiro/specs/ai-provider-model-picker/design.md:127`).
  **Do not use it.** Stay on native `@ai-sdk/*`.
- `@mastra/core/agent` cannot be loaded as-is in the unit-test workspace. `growi-agent.spec.ts`
  stubs the `Agent` class because the repo pins `p-map@4` for Mastra and Mastra imports `pMapSkip`
  (`growi-agent.spec.ts:8-17`). Loop-level behavior tests therefore need a spike (S0.4)
  before they are planned as unit tests.

### 2.4 Ollama OpenAI-compatibility facts (docs.ollama.com/api/openai-compatibility)

- `/v1/chat/completions` supports `tools`, `stream_options.include_usage`, `reasoning_effort`,
  `response_format`. **`tool_choice` is listed as unsupported.** So
  `toolChoice: 'required'` does **not** force a tool call on Ollama.
- An API key is not required for local servers (the value is ignored).
- `/v1/embeddings` exists. `/v1/responses` exists but is non-stateful and partial.
- Context window: Ollama's default context length is small, and the `/v1` API offers no
  per-request `num_ctx`. It must be set on the server (`OLLAMA_CONTEXT_LENGTH` env var or a
  Modelfile `PARAMETER num_ctx`) *(verify the default for the Ollama version in use)*. **Silent
  prompt truncation is a primary cause of "the local model forgot to use tools / exited early".**
  See §6.1 F4.

### 2.5 Elastic facts (official docs, fetched 2026-09-24)

- `dense_vector`: `index` defaults to `true`, `similarity` to `cosine`, `element_type` to
  `float`. Max `dims` 4096. **`dims` cannot be changed after mapping creation.** Default
  `index_options.type` is `int8_hnsw` on 9.0, and `bbq_hnsw` for dims ≥ 384 on 9.1+
  (`bbq_disk` on 9.4+ where licensed). The pre-8.11 default for `index` was different
  *(verify for the minimum 8.x minor you support)*. Set `index: true` and
  `similarity: 'cosine'` explicitly in the mapping so behavior does not depend on the minor.
- kNN filters are **pre-filters**: "The filter is applied during approximate kNN search to
  ensure that k matching documents are returned."
- Retrievers (`standard`, `knn`, `rrf`, `linear`, …) forbid top-level `query`, `knn`,
  `search_after`, `terminate_after`, **`sort`**, `rescore`. The chat tool sorts by
  `updatedAt` / `createdAt` on request, so retriever-based hybrid cannot serve those calls.
- `semantic_text`: **GA since 9.0** (earlier on 8.x as preview). It auto-chunks, supports
  custom `inference_id`, and supports highlighting. The docs say it typically requires "an
  appropriate license" because it calls the Inference API. Elastic's subscription matrix lists
  the Inference API, vector search, RRF, the linear retriever and ELSER in all tiers today. **Do
  not assert licensing for older 8.x minors from today's matrix *(verify per deployment)*.**
- The ES `openai` inference service accepts a custom `url` (so ES itself could call an
  OpenAI-compatible embeddings endpoint). A `custom` inference service also exists.

### 2.6 Codebase facts the design depends on

Provider layer (`apps/app/src/features/mastra/`):

| Fact | Location |
|---|---|
| Provider set is a static const map; `AiProvider` is its key union; `AI_PROVIDERS` / `mapProviders` iterate it | `interfaces/ai-provider.ts:33-72` |
| Non-enumerable providers (no models.dev catalog) get free-text model IDs and are excluded from `CATALOG_PROVIDERS` automatically | `interfaces/ai-provider.ts:11-14`, `server/services/ai-sdk-modules/chat-model-filter.ts:27-30` |
| Availability rule is a pure client-safe function with a hard-coded `azure-openai` branch | `interfaces/provider-availability-rule.ts:66-94` |
| Server availability adapter reads **azure** settings unconditionally for every provider | `server/services/ai-sdk-modules/llm-providers/provider-availability.ts:53-57` |
| Non-secret settings (`ai:providers`) and secrets (`ai:providerApiKeys`) are separate config keys; keys are write-only; GET returns only `isApiKeySet` | `interfaces/provider-settings.ts`, `server/service/config-manager/config-definition.ts:1313-1321`, `routes/admin-ai-settings/get-ai-settings.ts:153-169` |
| Config accessor normalizes each provider entry (trim, blank→undefined), azure-specific | `llm-providers/config.ts:68-98` |
| Resolver dispatch is a data-driven `Record<AiProvider, resolver>` | `llm-providers/index.ts:26-34` |
| PUT requires an entry for **every** provider (fixed-slot rule) and rebuilds `ai:providers` with an azure-only branch | `routes/admin-ai-settings/put-ai-settings.ts:169-176, 229-278` |
| Allow-list entry keys are whitelisted (`provider`, `modelId`, `providerOptions`, `isDefault`) | `routes/admin-ai-settings/validate-allowed-models.ts:10-15` |
| Env-only mode locks `app:aiEnabled`, `ai:providers`, `ai:providerApiKeys` | `config-definition.ts:1656-1666` |
| The UI panel has a two-way warning ternary (azure endpoint vs "missing API key") and an azure-only sub-form | `client/admin/ProviderPanel.tsx:112-123, 155-157` |
| Form values carry an azure object for every slot | `client/admin/ai-settings-form-values.ts:73-77, 119-126, 231-240` |
| Cache clear + dedup reset on save and on s2s `configUpdated` | `put-ai-settings.ts:445-449`, `server/services/model-config-sync.ts` |
| Swagger enums list the four providers literally | `get-ai-settings.ts:98`, `put-ai-settings.ts:107`, `get-available-models.ts:78` |

Chat agent:

| Fact | Location |
|---|---|
| Instructions ask to search first and read candidates; there is no minimum and no stop rule | `server/services/mastra-modules/agents/growi-agent.ts:13-23` |
| Tools are registered under keys `fullTextSearchTool` and `getPageContentTool`. The **client types key off these names** (`tool-<key>` parts) | `growi-agent.ts:48-51`, `interfaces/chat-tools.ts` |
| `maxSteps: 10`, no `prepareStep`, no `toolChoice`, no `onIterationComplete` | `server/routes/post-message.ts:98-111` |
| The "Stream finished" log already records `finishReason`, `stepCount`, and token usage | `post-message.ts:173-183` |
| Search tool: default 10 hits, max 20; returns `pageId`, `pagePath`, optional `snippet`; resolves user groups the same way as `/_search` | `tools/full-text-search-tool.ts:35-42, 139-160` |
| Page tool: outline first; `limit` default 200 lines, max 500; viewer-checked by `findByIdAndViewer` / `findByPathAndViewer` | `tools/get-page-content-tool.ts:109-116, 213-223` |
| Memory keeps the last 30 messages (context pressure for small local models) | `mastra-modules/memory/index.ts:29-32` |
| Suggest-path already has the budget-wrapper pattern (`limitedSearchTool` with a mutable per-request `SearchBudget` in `RequestContext`) | `agents/suggest-path/limited-search-tool.ts`, `agents/suggest-path/request-context.ts` |
| Suggest-path's `maxSteps` is derived from its budgets | `features/ai-tools/suggest-path/server/services/engines/agentic-engine.ts` (`computeMaxSteps`) |

Search / Elasticsearch (`apps/app/src/server/service/`):

| Fact | Location |
|---|---|
| Mappings define `body_embedded: { type: 'dense_vector', dims: 768 }` in both ES8 and ES9 | `search-delegator/mappings/mappings-es{8,9}.ts:73-76` |
| `prepareBodyForCreate` copies `page.revisionBodyEmbedded` into `body_embedded` | `search-delegator/elasticsearch.ts:697-725` |
| The aggregation pipeline never projects `revisionBodyEmbedded`, so the vector field is always absent | `search-delegator/aggregate-to-index.ts:120-157`, `bulk-write.d.ts:29` |
| Every index write is a **full-document `index` op** from the aggregation, including writes triggered by bookmark, comment, tag, and **`addSeenUsers` (page view)** events | `search.ts:202-311`, `elasticsearch.ts:782-892` |
| Bulk item errors are only logged (`errors=${bulkResponse.errors}`). A document rejected by ES (e.g. a wrong-dimension vector) silently disappears from **keyword** search too | `elasticsearch.ts:850-866` |
| Rebuild = create tmp index → async `_reindex` old→tmp → swap alias → delete+recreate main → `addAllPages` from Mongo | `elasticsearch.ts:359-423`, `es9-client-delegator.ts:80-89` |
| The ES9 search path forwards **only** `query`, `sort`, `highlight` from `body`; anything else (e.g. `knn`) is dropped | `elasticsearch.ts:1024-1034` |
| Permission filter = `bool.filter` → `bool.should` of grant conditions, driven by `security:list-policy:hideRestrictedByOwner/ByGroup` | `elasticsearch.ts:1331-1398` |
| `formatSearchResult` re-loads pages by `_id $in` **without** a viewer check. The ES filter is the **only** listing gate | `search.ts:758-865` (`findPageListByIds` at `search.ts:94-117`) |
| `disableUserPages` is applied as a `not_prefix` term at parse time, so it ends up in `bool.filter` | `search.ts:441-479` |
| Snippets come only from `_highlight`, and are gated by `canShowSnippet` | `search.ts:820-842, 867-896` |
| ES integration tests use a **live** cluster when `ELASTICSEARCH_URI` is set (`describe.skipIf(!hasElasticsearch)`) | `search-delegator/elasticsearch.integ.ts:10-13` |

---

## 3. Process: which specs to open

The repo is spec-driven (`.kiro/specs/`, the `kiro-*` skills). Related completed specs, all
`"phase": "implementation-complete"`, `language: ja`:

- `ai-provider-multi-vendor`: fixed four-slot provider model, `ai:providers` +
  `ai:providerApiKeys`, availability rule, PUT "all four entries required".
- `ai-provider-multi-model`: allow-list, default model.
- `ai-provider-model-picker`: bundled models.dev catalog, the explicit rejection of Mastra's
  model router.
- `ai-agentic-search`: the chat agent's search/read tools.
- `suggest-path`: the suggest-path agent and its budgets.

**Objective 1 changes the contract of `ai-provider-multi-vendor`**: the provider set grows,
"all four" becomes "all five", and there is a new availability reason. Per
`.claude/rules/spec-lifecycle.md` that makes it an **amend spec**:

- Name it e.g. `ai-provider-openai-compatible`.
- `brief.md` and `design.md` carry an **"Amend target"** section listing
  `ai-provider-multi-vendor` (provider set, availability rule, PUT completeness rule, admin
  UI) and `ai-provider-model-picker` (non-enumerable provider handling), and naming any
  touched Revalidation Triggers.
- The **final task** in its `tasks.md` is the port-back-and-delete task from the rule's
  template. Append new requirement IDs; never renumber.

**Objective 2** is new behavior for the `ai-agentic-search` feature. It either extends that spec
or opens a new spec (e.g. `ai-chat-research-policy`). If it changes an approved requirement
of `ai-agentic-search` (for example, if that spec fixed `maxSteps: 10`), treat it as an amend
spec too. Check `ai-agentic-search/requirements.md` first.

**Objective 3** is new functionality: a new spec (e.g. `search-semantic-retrieval`). It is not an amend spec,
unless it changes `search-filters` behavior (it should not).

Spec documents are written in the language set in `spec.json` (`ja` for the AI specs). This
`plan.md` is not a spec, and is in English.

---

## 4. Phase 0: spikes (throwaway, no product code)

Purpose: confirm the handful of assumptions that, if wrong, would change the design. Put
scratch code **outside the repo**, or in a new directory such as `tmp/spikes/` that you add to
`.git/info/exclude` first. The repo-root `tmp/` is **not** ignored as a whole; only
`tmp/memory-profiler/` and `tmp/memory-leak-investigation/` are, per `.gitignore:59-60`. **Do
not commit spike code.** Record each result as a short note in the relevant spec's
`research.md`. The same applies to the `tmp/eval/` files mentioned in §8.

### S0.1 Endpoint behavior matrix (curl only)

For every server you intend to support, run the three requests below and record the answers
in a table (server, version, model, answers).

1. Chat with a tool, `tool_choice` omitted:
   ```bash
   curl -s $BASE/chat/completions -H 'Content-Type: application/json' \
     -H "Authorization: Bearer ${KEY:-placeholder}" -d '{
       "model": "'$MODEL'",
       "messages": [{"role":"user","content":"Find pages about deployment. Use the tool."}],
       "tools": [{"type":"function","function":{"name":"search","description":"search the wiki",
         "parameters":{"type":"object","properties":{"query":{"type":"string"}},"required":["query"]}}}]
     }' | jq '.choices[0].message'
   ```
   Record: does `tool_calls` appear, or is the call written as text in `content` (for example
   `<tool_call>{...}</tool_call>` or raw JSON)?
2. Same request with `"tool_choice": "required"`. Record: honored / ignored / 400 error.
3. Same request with `"stream": true, "stream_options": {"include_usage": true}`. Record:
   does the final chunk carry `usage`? Does streaming of `tool_calls` deltas work?

Servers to cover at minimum:

| Server | Base URL form | Tool-calling prerequisite to check |
|---|---|---|
| Ollama | `http://ollama:11434/v1` | Model must list the `tools` capability (`ollama show <model>`). `tool_choice` is expected to be ignored |
| vLLM | `http://vllm:8000/v1` | Server started with `--enable-auto-tool-choice --tool-call-parser <parser>` (for example `hermes` for Qwen). Without these, tool calls come back as text |
| llama.cpp `llama-server` | `http://host:8080/v1` | Started with `--jinja` (and a template with tool support) |
| LM Studio | `http://host:1234/v1` | Model loaded with tool-use support |

Expected outcome: a table that feeds §6.4 (which enforcement levers work where) and the admin
help text in §5.5.

### S0.2 (Optional) baseline measurement before Phase 1 exists

To measure today's early-exit behavior with a local model **before** Objective 1 lands, you can
exploit §2.2 fact 3. Set `OPENAI_BASE_URL=http://ollama:11434/v1` in the app environment,
enable the `openai` provider with any dummy key, and put the local model in
`AI_ALLOWED_MODELS` (the env path bypasses the admin UI's catalog `<select>`). The `openai`
provider uses the **Responses** API, which Ollama supports only partially, so results are
indicative only. Use this for a quick S0 baseline in §8. **Never ship or document it.** OD-6 may
remove the fallback.

### S0.3 Mastra loop mechanics (scratch script against the real packages)

After `pnpm install`, write a scratch script in `tmp/` that builds a real `Agent` from
`@mastra/core/agent` with a scripted mock model (`MockLanguageModelV3` from `ai/test`, or
a hand-written `LanguageModelV2` object whose `doStream` returns scripted chunks) and two
dummy tools. Verify each point below and write the answer down:

1. `prepareStep` returning `{ toolChoice: 'none' }` → the mock's `doStream` call options
   contain **no tools**.
2. `prepareStep` returning `{ activeTools: ['search'] }` → only `search` is sent.
3. `prepareStep` receives `stepNumber` 0, 1, 2… and `steps[i].toolCalls` / `toolResults`
   with the tool **registration keys** as `toolName`.
4. `prepareStep` returning `systemMessages: [...args.systemMessages, extra]`: does `extra`
   stay in the message list for later steps? (`replaceAllSystemMessages` suggests yes.) This
   decides whether the nudge helper must de-duplicate (§6.4.4 design already de-duplicates).
5. `onIterationComplete` returning `{ continue: true }` when `isFinal && text === ''` → the
   loop runs one more step, and `maxSteps` is still respected.
6. The `context: [{ role: 'system', content: '...' }]` stream option: the message reaches the
   model, and it is **not** written to the memory thread (use the real `MongoDBStore` against
   `mongo:27017` with a throwaway database name, then read the thread's messages back).
7. `toAISdkStream(stream, { from: 'agent', version: 'v6' })` given an `is-task-complete`
   chunk: dropped, or forwarded as an unknown part? (Only needed if strategy S6 in §8 is
   pursued.)
8. **Off-by-one check for the answer guarantee (R2).** With `maxSteps = N` and a mock that
   *always* returns a tool call: the N-th LLM call must carry **no tools**, and the run must end
   with text and `finishReason: 'stop'`. If Mastra counts steps differently (e.g. the N-th step
   never happens), adjust R2's threshold (`maxSteps - 1` vs `maxSteps - 2`) and `computeChatMaxSteps`
   before writing product code. The "F3 = 0 %" target in §8.1 rests on this.
9. **Removed-tool call.** With `activeTools: ['getPageContentTool']`, script the mock to call
   `fullTextSearchTool` anyway (weak models do this from history). Record what happens: does
   Mastra execute it (then the budget wrapper must return `limit_exceeded`, which it does), or
   does it produce a tool error result, or throw? The wrappers must behave correctly in every case.
10. **Live provider check: tool history without tool definitions** (one real call per provider
    family: OpenAI or Azure, Anthropic, Google, Ollama, vLLM). Send a 3-message history
    (user → assistant tool call → tool result) with `toolChoice: 'none'` through a real Mastra
    `Agent`, which strips the tools. Some APIs reject requests whose history contains tool calls
    but which define no tools. Anthropic historically returned 400 in this case *(verify
    today's behavior)*. Record per provider: accepted / 400. **If any provider rejects it**, R2,
    R4, R5 and R7 cannot use `toolChoice: 'none'` for that provider. Fall back to *keeping the
    tools* and relying on the wrappers' `limit_exceeded` + guidance, and reserve one extra step
    in `computeChatMaxSteps`. Make that a per-provider **declared** capability (data, not an
    `if (provider === …)` in the policy; e.g. a `supportsToolFreeFollowUp` flag next to
    `AI_PROVIDER_DEFS` metadata) and state the weaker guarantee for that provider in the spec.

### S0.4 Test-harness feasibility for loop-level tests

Try to construct the real `Agent` inside a Vitest **integration** test (`*.integ.ts`). If it
fails because of the `p-map@4` override (`pMapSkip` missing), loop behavior is covered by
(a) pure-function unit tests of the policy (§6.6) and (b) the manual/eval harness (§8), **not**
by unit tests of Mastra internals. Record the outcome in the research note so nobody retries it
blindly.

### S0.5 Elasticsearch kNN mechanics (devcontainer ES 9.3.3 plus the minimum 8.x you support)

Using a throwaway index (never the GROWI alias):

1. Create an index with `body_embedded: { type: 'dense_vector', dims: 4, index: true, similarity: 'cosine' }`,
   plus `grant` (integer) and `granted_groups` (keyword). Index 6 docs with hand-made vectors.
2. Run a top-level `knn` search **with** `knn.filter` = the grant bool, and confirm that
   restricted docs never appear, even when they are the nearest neighbors.
3. Run the same as a `knn` **query** inside `bool: { must: [knn], filter: [grant bool] }`.
   Does the filter act as a pre-filter (still `k` results) or a post-filter (fewer results)?
   The design uses the top-level `knn.filter` form regardless, so this is informational.
4. Add `highlight: { fields: { body: { highlight_query: { match: { body: 'deploy' } } } } }` to
   the kNN request and check that highlights come back for kNN-only hits.
5. Confirm `_source: [...]` without `body_embedded` keeps vectors out of responses.
6. Repeat 1–5 on the lowest ES 8 minor you intend to support (for example
   `docker run -e discovery.type=single-node -e xpack.security.enabled=false docker.elastic.co/elasticsearch/elasticsearch:8.x`).
   Record the minimum version at which all of it works. That becomes the semantic-mode version
   gate (§7.6.5).

### S0.6 Embedding model quick check (Japanese + English)

With Ollama, pull 2–3 candidate embedding models, for example `nomic-embed-text` (768-d,
mostly English), `embeddinggemma` (768-d, multilingual) and `bge-m3` (1024-d, multilingual).
Embed ~20 short JA/EN sentence pairs and a few paraphrases, and check the cosine ranking.
Record dims, latency per batch of 16, and the max input tokens. This picks the default model
and dims for §7.4.

---

## 5. Objective 1: custom OpenAI-compatible model endpoint

### 5.1 How a chat model is resolved today (read this before changing anything)

```
Admin UI (AiSettings.tsx, react-hook-form)
  └─ PUT /_api/v3/ai-settings  (put-ai-settings.ts)
       ├─ validators: providers must contain ALL AI_PROVIDERS keys; allowedModels invariants
       ├─ writes ai:providers        (non-secret Record<AiProvider, {enabled, azureOpenaiSettings?}>)
       ├─ writes ai:providerApiKeys  (secret, merge-only, never cleared, trimmed)
       ├─ writes ai:allowedModels    ([{provider, modelId, providerOptions?, isDefault?}])
       └─ clearResolvedMastraModelCache() + clearAvailabilityLogDedup()   (+ s2s on other nodes)

Chat request (post-message.ts)
  └─ resolveEffectiveModelKey(clientKey)            effective-model-key.ts
        └─ getAvailableModels()                      provider-availability.ts
              ├─ getAvailableProviders() → evaluateProviderAvailability()  (pure rule)
              └─ getAllowedModels() filtered by available providers
  └─ growiAgent.model = resolveMastraModel(key)     resolve-mastra-model.ts
        └─ modelResolvers[provider](modelId)        llm-providers/index.ts  (dynamic import of @ai-sdk/*)
```

Properties the new provider must keep:

- **Secrets never leave the server.** GET returns `isApiKeySet` only. Error messages name
  the provider and the env var, never the value.
- **Fail soft on malformed config.** Accessors read through `unknown`, normalize, and warn
  once through `warn-dedup`.
- **Lazy SDK loading.** `no-eager-provider-imports.spec.ts` fails if any `@ai-sdk/*` package
  becomes statically reachable from the provider barrel or the Mastra instance.
- **Env-only mode.** Connection settings (`ai:providers`, keys, the AI toggle) are read-only
  when `env:useOnlyEnvVars:ai` is on; `allowedModels` stays editable.
- **Allow-list is the single checkpoint.** A client-supplied model key is never trusted.

### 5.2 Scope decision: one slot, not N named endpoints (D1)

| | One fixed slot `openai-compatible` (default) | N named endpoints |
|---|---|---|
| Data model | Adds one key to `AI_PROVIDER_DEFS`. `ai:providers` stays a Record keyed by `AiProvider` | `AiProvider` becomes open-ended (e.g. `openai-compatible:<slug>`); `ai:providers` needs an array or a Record with dynamic keys |
| ModelKey `${provider}/${modelId}` | Unchanged | Slug must be `/`-free; `parseModelKey` + `isAiProvider` become dynamic (config-dependent) checks, used on the client too |
| Client-safe static metadata (`getProviderLabel`, `AI_PROVIDER_DEFS`) | Unchanged pattern | Must be served by an API; labels become admin-defined |
| PUT completeness rule ("all providers present") | "all five" | Replaced by add/remove semantics |
| Swagger / validators / tests | Mechanical updates | Redesign |
| Covers | "One Ollama / vLLM / gateway endpoint" (the stated need) | Several local servers at once (e.g. one GPU box per model) |

The prior spec already recorded this fork (`.kiro/specs/ai-provider-multi-vendor/research.md:163`:
allowing two OpenAI-compatible endpoints requires "an array plus unique registration names",
a requirements-level decision). **Default: one slot.** A single OpenAI-compatible gateway
(LiteLLM, or vLLM serving several models) already serves multiple models behind one base URL,
which covers most multi-model needs. §5.11 sketches N endpoints for later.

### 5.3 SDK choice (D2)

Use `createOpenAI({ baseURL, apiKey, name: 'openai-compatible' }).chat(modelId)` from the
already-installed `@ai-sdk/openai`:

- **Pros:** zero new dependency. Same `OpenAIChatLanguageModel` class that
  `@ai-sdk/azure` uses, so `providerOptions` behave exactly as the azure provider's do (namespace
  `openai`). Lazy dynamic import pattern identical to `openai.ts`.
- **Cons:** no parsing of `reasoning_content`/`reasoning` (thinking is dropped; final answers
  are unaffected); always sends an `Authorization` header (a harmless placeholder when keyless);
  model-id-based reasoning heuristics (overridable through `forceReasoning` /
  `systemMessageMode`).
- **Rejected:** Mastra `OpenAICompatibleConfig` (model router; rejected by prior spec).
  `@ai-sdk/openai-compatible` stays an optional follow-up if displaying reasoning from local
  models becomes a requirement.

### 5.4 Data model and config

New client-safe interface file
`apps/app/src/features/mastra/interfaces/openai-compatible-config.ts`:

```ts
/**
 * Connection settings for the 'openai-compatible' provider entry inside ai:providers.
 * Non-secret: the optional API key lives in ai:providerApiKeys like every other provider.
 * `baseURL` includes the version path (e.g. http://ollama:11434/v1); the SDK appends
 * /chat/completions and /embeddings.
 */
export interface OpenaiCompatibleConfig {
  baseURL?: string;
  /**
   * Admin attestation that every model allowed under this provider supports tool
   * calling (GROWI chat cannot work without it). Catalog-less providers carry no
   * tool_call metadata, so availability requires this to be true.
   */
  toolCallingConfirmed?: boolean;
}
```

Extend (types only, all optional, so no data migration is needed):

- `interfaces/provider-settings.ts`: `AiProviderSettings.openaiCompatibleSettings?: OpenaiCompatibleConfig`.
- `interfaces/ai-settings.ts`: `AiProviderStatus.openaiCompatibleSettings?`,
  `AiProviderUpdateRequest.openaiCompatibleSettings?`. Update the doc comments ("all 4" → "all").
- `interfaces/ai-provider.ts`: add
  `'openai-compatible': { enumerable: false, label: 'OpenAI-compatible' }` **last** (insertion order
  drives tab order; keep the existing four first).

Config keys: **no new keys.** The provider lives inside the existing `ai:providers` /
`ai:providerApiKeys` / `ai:allowedModels`, so env-only mode, s2s cache invalidation and the
secret split all apply automatically. Env example (JSON, single-quoted for the shell):

```bash
AI_ENABLED=true
AI_PROVIDERS='{"openai-compatible":{"enabled":true,"openaiCompatibleSettings":{"baseURL":"http://ollama:11434/v1","toolCallingConfirmed":true}}}'
AI_ALLOWED_MODELS='[{"provider":"openai-compatible","modelId":"qwen3:14b","isDefault":true}]'
# AI_PROVIDER_API_KEYS='{"openai-compatible":"sk-..."}'   # only for endpoints that need a key
```

**Keyless (D3):** the API key is optional for this provider. The resolver uses
`getApiKey('openai-compatible') ?? OPENAI_COMPATIBLE_KEYLESS_PLACEHOLDER`, where the placeholder
is a constant such as `'growi-no-api-key'`. It is not a secret, it prevents the SDK's
`OPENAI_API_KEY` env fallback, and servers that ignore keys ignore it.

### 5.5 Availability rule (pure, shared by server and client)

`interfaces/provider-availability-rule.ts`:

```ts
export type ProviderUnavailableReason =
  | 'disabled'
  | 'missing-api-key'
  | 'missing-azure-endpoint'
  | 'missing-base-url'              // new: openai-compatible without baseURL
  | 'tool-calling-unconfirmed';     // new: openai-compatible without attestation

export interface ProviderAvailabilityInput {
  readonly provider: AiProvider;
  readonly enabled: boolean;
  readonly hasApiKey: boolean;
  readonly azureOpenaiSettings?: Readonly<Pick<AzureOpenaiConfig, 'resourceName' | 'baseURL' | 'useEntraId'>>;
  readonly openaiCompatibleSettings?: Readonly<OpenaiCompatibleConfig>;
}
```

Evaluation order for `openai-compatible` mirrors the resolver's real throw order (the same
reasoning the azure branch documents):

1. `!enabled` → `disabled`
2. `!isNonBlankString(baseURL)` → `missing-base-url`
3. `toolCallingConfirmed !== true` → `tool-calling-unconfirmed`
4. otherwise `available` (the key is **not** required)

Put this in its own small branch next to the azure branch. Two special providers do not yet
justify a metadata abstraction. If a third appears, move key and endpoint requirements into
`AI_PROVIDER_DEFS` metadata.

Server adapter `llm-providers/provider-availability.ts`:

- Read the **provider's own** settings: `const settings = getProviderSettings(provider)`,
  then pass `azureOpenaiSettings: settings?.azureOpenaiSettings` and
  `openaiCompatibleSettings: settings?.openaiCompatibleSettings`. Today line 56-57 reads the
  azure entry for every provider. It is harmless, but it becomes misleading with two special
  providers.
- Widen `warnMisconfigured`'s `reason` parameter to
  `Exclude<ProviderUnavailableReason, 'disabled'>`.

### 5.6 Tool-calling guarantee (D4)

Catalog providers are filtered by models.dev's `tool_call` flag at vendoring time
(`chat-model-filter.ts:55-56`). Catalog-less providers have no metadata. `azure-openai` today
accepts any deployment name with **no** tool-calling check at all. The minimal design gives
the new provider **stronger** protection than azure has today:

1. **Attestation (enforced):** `toolCallingConfirmed` must be `true` for the provider to be
   available. Until then its models are excluded from chat (same mechanism as a missing key).
   The admin UI shows a warning that explains why.
2. **Probe (diagnostic, admin-only):** a "Test model" action next to each allowed-model row
   of catalog-less providers (and optionally all providers) calls
   `POST /_api/v3/ai-settings/test-model` with `{ provider, modelId }`. The server:
   - requires `SCOPE.WRITE.ADMIN.AI` + login + admin (it makes an outbound, possibly billed call);
   - **uses the saved connection config only** (never a URL or key from the request body, so
     the route cannot be used as a generic SSRF probe);
   - builds the model through `modelResolvers[provider](modelId)` (bypassing allow-list rounding,
     so an admin can test before adding the model to the list);
   - runs `generateText` from `ai` with one dummy tool (`ping({ value: string })`), a short
     prompt instructing the model to call it, `toolChoice: 'required'`, `maxOutputTokens: 64`,
     `maxRetries: 0`, and a 20 s `AbortSignal.timeout`;
   - returns `{ reachable, toolCallObserved, errorKind?, message? }`, where `errorKind` is one of
     `unauthorized | not-found | tools-unsupported | timeout | connection | other`, and `message`
     is sanitized through the same rule as `resolveChatErrorMessage` (one line, provider-authored
     text only, no URL/body/stack).
   - Note for the help text: on Ollama `tool_choice` is ignored, so `toolCallObserved: false`
     from a tool-capable model is possible. Tool-*incapable* Ollama models fail with HTTP 400
     ("does not support tools"), which maps to `tools-unsupported`.

The probe stays advisory in the default scope. The stricter optional variant (persisting a
verification record keyed by `(baseURL, modelId)` and requiring it for selection) is OD-2.

### 5.7 Resolver

New file `llm-providers/openai-compatible.ts`:

```ts
import type { MastraModelConfig } from '@mastra/core/llm';

import { getApiKey, getProviderSettings } from './config';

// Sent when no key is stored: prevents @ai-sdk/openai from falling back to
// process.env.OPENAI_API_KEY (which would leak a real key to a custom host).
// Keyless servers (e.g. local Ollama) ignore the Authorization header.
const KEYLESS_PLACEHOLDER = 'growi-no-api-key';

export const resolveOpenaiCompatibleModel = async (
  modelId: string,
): Promise<MastraModelConfig> => {
  const baseURL =
    getProviderSettings('openai-compatible')?.openaiCompatibleSettings?.baseURL;
  if (baseURL == null) {
    throw new Error(
      'The OpenAI-compatible provider requires a base URL (set it via the admin AI settings or the AI_PROVIDERS environment variable)',
    );
  }
  const apiKey = getApiKey('openai-compatible') ?? KEYLESS_PLACEHOLDER;
  const { createOpenAI } = await import('@ai-sdk/openai');
  // .chat(): Chat Completions. The default call would build a Responses-API model,
  // which most OpenAI-compatible servers do not fully implement.
  return createOpenAI({ baseURL, apiKey, name: 'openai-compatible' }).chat(modelId);
};
```

Register it in `llm-providers/index.ts` (`'openai-compatible': resolveOpenaiCompatibleModel`).
The `Record<AiProvider, …>` type makes a missing entry a compile error, which is intended.

Config accessor `llm-providers/config.ts`:

- `normalizeOpenaiCompatibleSettings(value: unknown): OpenaiCompatibleConfig | undefined`:
  `baseURL: asHttpUrl(value.baseURL)` and `toolCallingConfirmed: asOptionalBoolean(...)`.
- `asHttpUrl`: `asNonBlankString`, then `new URL()` succeeds, protocol `http:`/`https:`, no
  `username`/`password`. Otherwise `undefined` plus
  `warnOnce('ai:providers|openai-compatible-base-url-invalid', …)`. **Never log the value.** It
  may contain credentials if hand-edited.
- Add `openaiCompatibleSettings` to `normalizeProviderSettings`.
- `readProviderApiKeys` already filters by `isAiProvider`, so the new key is accepted
  automatically once `AI_PROVIDER_DEFS` contains it.

### 5.8 Admin API changes

`routes/admin-ai-settings/put-ai-settings.ts`:

1. Validation: extend `isValidProviderEntryRequest` with
   `value.openaiCompatibleSettings == null || isValidOpenaiCompatibleSettingsRequest(value.openaiCompatibleSettings)`:
   a record; `baseURL` optional string that, **when non-blank**, passes the shared
   `isValidEndpointUrl` (below); `toolCallingConfirmed` optional boolean.
2. Shared URL rule `apps/app/src/features/mastra/utils/endpoint-url.ts` (client-safe, pure,
   used by the server validator, the config accessor, and the form's inline validation):
   ```ts
   export const isValidEndpointUrl = (value: string): boolean => {
     try {
       const url = new URL(value.trim());
       return (url.protocol === 'http:' || url.protocol === 'https:')
         && url.username === '' && url.password === ''
         && url.hash === '';
     } catch { return false; }
   };
   ```
   Reject credentials in the URL because `baseURL` is **non-secret** and is returned by GET. The
   error message must tell the admin to put the key in the API key field instead.
3. Building `ai:providers`: replace the `provider === 'azure-openai'` ternary with a
   **declared map** (coding-style "data-driven control"):
   ```ts
   const CONNECTION_SETTINGS_BUILDERS: Partial<Record<AiProvider,
     (entry: AiProviderUpdateRequest) => Partial<AiProviderSettings>>> = {
     'azure-openai': (e) => optional('azureOpenaiSettings', buildAzureOpenaiConfig(e.azureOpenaiSettings)),
     'openai-compatible': (e) => optional('openaiCompatibleSettings', buildOpenaiCompatibleConfig(e.openaiCompatibleSettings)),
   };
   // config[provider] = { enabled: entry.enabled === true, ...(CONNECTION_SETTINGS_BUILDERS[provider]?.(entry) ?? {}) };
   ```
   `buildOpenaiCompatibleConfig` trims `baseURL`, drops blanks, keeps `toolCallingConfirmed`
   only when `true`, and collapses an empty object to `undefined` (same as azure).
4. Swagger: add the `openaiCompatibleSettings` schema; update the enum lists (search for
   `enum: [openai, anthropic, google, azure-openai]` in `get-ai-settings.ts`,
   `put-ai-settings.ts`, `get-available-models.ts`) and the prose "all four".

`routes/admin-ai-settings/get-ai-settings.ts`: `buildProviderStatus` returns
`openaiCompatibleSettings` only on that entry. Use the same map approach
(`CONNECTION_SETTINGS_STATUS_PICKERS`) or one extra branch; keep it symmetric with the PUT.

New route `routes/admin-ai-settings/post-test-model.ts` (D4 probe), mounted in
`routes/admin-ai-settings/index.ts` as `router.post('/test-model', postTestModelFactory(crowi))`,
with the middleware chain of `putAiSettingsFactory`: scope WRITE, login, admin, `addActivity`,
express-validator (`provider` is an `AiProvider`; `modelId` is a non-blank trimmed string ≤
`MAX_MODEL_KEY_LENGTH`), `apiV3FormValidator`. Keep the probe logic in a pure-ish service
`server/services/ai-sdk-modules/probe-model.ts` that takes an already-built model (executor
receives its input), so it can be unit-tested with a mock model. The route only adapts it.

### 5.9 Admin UI changes

Files under `apps/app/src/features/mastra/client/admin/`:

| File | Change |
|---|---|
| `ai-settings-form-values.ts` | `ProviderFormValue.openaiCompatibleSettings: Required<OpenaiCompatibleConfig>` (controlled defaults `''` / `false`); seed in `toFormValues`; attach in `toProviderUpdate` for that provider only (map, like the server); pass it into `evaluateFormProviderAvailability` |
| `OpenaiCompatibleSettings.tsx` (new) | Mirrors `AzureOpenaiSettings.tsx`: base URL input with inline validation via `isValidEndpointUrl`; help text listing example URLs (Ollama/vLLM/LM Studio/llama.cpp, "include `/v1`"); `toolCallingConfirmed` switch with an explanation; a note that the API key is optional for local servers; everything disabled under env-only |
| `ProviderPanel.tsx` | Replace the azure-only render with a declared map `PROVIDER_CONNECTION_SECTIONS: Partial<Record<AiProvider, ComponentType<{ disabled: boolean }>>>`. Replace the two-way warning ternary with a map `Record<Exclude<ProviderUnavailableReason, 'disabled'>, i18nKey>`. The current ternary would show "missing API key" for the new reasons, which would be wrong. API key help text per provider (optional wording for `openai-compatible`) |
| `AllowedModelsField.tsx` / `AllowedModelRow.tsx` | Free text is automatic (empty catalog). Replace `isAzure` label switching with a map of per-provider label keys (azure → deployment name; openai-compatible → "Model ID as served by the endpoint, e.g. `qwen3:14b`"). Add the "Test model" button for catalog-less providers, disabled while the provider's connection fields are dirty (the probe uses **saved** config), showing the probe result inline |
| `provider-options-namespace.ts` | `'openai-compatible': 'openai'` |
| `ProviderTabs.tsx` | Nothing structural (iterates `AI_PROVIDERS`); the status dot uses the shared rule |

i18n (`apps/app/public/static/locales/{en_US,fr_FR,ja_JP,ko_KR,zh_CN}/admin.json`), new keys
under `ai_settings.`:

```
openai_compatible_section_title
openai_compatible_base_url_label
openai_compatible_base_url_help          (examples; "include /v1"; credentials not allowed in the URL)
openai_compatible_base_url_invalid
openai_compatible_tool_calling_label
openai_compatible_tool_calling_help      (why: chat requires tools; how to check on Ollama/vLLM)
openai_compatible_api_key_help           (optional for local servers)
openai_compatible_model_id_label
openai_compatible_add_model
provider_warning_missing_base_url
provider_warning_tool_calling_unconfirmed
test_model_button / test_model_running / test_model_ok / test_model_no_tool_call
test_model_error_unauthorized / _not_found / _tools_unsupported / _timeout / _connection / _other
```

Add every key to all five locales in the same PR. English text may be temporarily reused
for fr/ko/zh if that is the repo's practice; check `.kiro/specs/i18n-key-audit` for any test
that enforces key parity.

### 5.10 Tasks (Objective 1)

Each task is ≤ ½ day. Do them in order; each ends green.

**T1.1 Provider identity.**
- Files: `interfaces/ai-provider.ts`, `interfaces/ai-provider.spec.ts`.
- Add the `openai-compatible` entry (non-enumerable, label "OpenAI-compatible").
- Fix every compile error the new union member causes. Expected: `llm-providers/index.ts`
  (resolver map, filled in T1.4), `client/admin/provider-options-namespace.ts` (namespace map).
  Put a temporary resolver stub in place only if needed to keep the build green inside this task.
- Tests: update "enumerates exactly the four supported vendors" to five; `isAiProvider('openai-compatible')`;
  `getProviderLabel`; `CATALOG_PROVIDERS` still excludes it (`chat-model-filter.spec.ts`);
  `persistedModelCatalogSchema` unaffected (`build-model-catalog.spec.ts` stays green).

**T1.2 Settings types + config accessor.**
- Files: `interfaces/openai-compatible-config.ts` (new), `interfaces/provider-settings.ts`,
  `interfaces/ai-settings.ts`, `utils/endpoint-url.ts` (new) + `endpoint-url.spec.ts`,
  `llm-providers/config.ts` + `config.spec.ts`.
- Tests (`config.spec.ts`): a valid URL is kept (trimmed); blank → undefined;
  `ftp://…` → undefined + warn once; `http://user:pass@host/v1` → undefined + warn whose
  message does **not** contain `user` or `pass`; non-object settings → undefined; the
  `toolCallingConfirmed` string `"true"` → undefined (strict boolean).
- Tests (`endpoint-url.spec.ts`): http/https accepted; path kept; credentials, hash, non-URL,
  and `javascript:` rejected.

**T1.3 Availability rule.**
- Files: `interfaces/provider-availability-rule.ts` + spec; `llm-providers/provider-availability.ts` + spec.
- Tests (table-driven): disabled → `disabled`; enabled + no baseURL → `missing-base-url` (even
  with a key); baseURL + not confirmed → `tool-calling-unconfirmed`; baseURL + confirmed + no
  key → available; azure cases unchanged; key-based providers unchanged. Server adapter:
  reads the provider's own settings; the warn is emitted once per (provider, reason) and
  never for `disabled`.

**T1.4 Resolver.**
- Files: `llm-providers/openai-compatible.ts` (new) + `openai-compatible.spec.ts`,
  `llm-providers/index.ts`, `llm-providers.spec.ts`.
- Tests: mock `@ai-sdk/openai` with `vi.mock` returning a `createOpenAI` spy whose result
  has a `chat` spy. Assert that `createOpenAI` is called with the stored `baseURL` and key;
  that `apiKey` is the **placeholder, never `undefined`,** when no key is stored; that
  `.chat(modelId)` is used and the provider function itself is **not** called; and that a
  missing baseURL throws **before** the dynamic import (assert `createOpenAI` was not called)
  with a message that has no key material.
- `no-eager-provider-imports.spec.ts` must stay green (dynamic import only).

**T1.5 PUT/GET.**
- Files: `put-ai-settings.ts` + spec, `get-ai-settings.ts` + spec, `index.spec.ts`,
  `env-only-mode.spec.ts`, `validate-allowed-models.spec.ts`.
- Tests: a providers payload missing `openai-compatible` → 400 (completeness);
  invalid/credentialed baseURL → 400 with the field flagged; a valid payload stores
  normalized `openaiCompatibleSettings` only under that provider; a blank key keeps the stored
  key (existing merge rule, now five providers); GET returns `openaiCompatibleSettings` only
  on that entry and never an `apiKey` field (assert with `not.toHaveProperty('apiKey')` on every
  entry); env-only still rejects `providers`; allowed models accept provider
  `openai-compatible` with a free-text id such as `qwen3:14b` or `org/model:tag`.

**T1.6 Probe service + route.**
- Files: `server/services/ai-sdk-modules/probe-model.ts` + spec, `routes/admin-ai-settings/post-test-model.ts`
  + spec, `routes/admin-ai-settings/index.ts`.
- Tests (service): a mock language model scripted to emit a tool call → `{ reachable: true, toolCallObserved: true }`;
  scripted plain text → `toolCallObserved: false`; `APICallError` with status 401 → `unauthorized`;
  400 with a "does not support tools" body → `tools-unsupported`; abort → `timeout`; a
  connection error → `connection`; `message` never contains the URL, the request body, or the key.
- Tests (route): non-admin → 403; the handler ignores any `baseURL`/`apiKey` in the body (assert
  the resolver was called with only `modelId`); an unknown provider → 400.

**T1.7 Form values.**
- Files: `ai-settings-form-values.ts` + spec.
- Tests: `toFormValues` seeds controlled defaults for the new slot; `buildUpdateRequest` sends
  five provider entries and attaches `openaiCompatibleSettings` only to that entry; env-only
  omits providers; `evaluateFormProviderAvailability` matches the server rule for each reason.

**T1.8 UI components.**
- Files: `OpenaiCompatibleSettings.tsx` (new) + spec, `ProviderPanel.tsx` + spec,
  `AllowedModelsField.tsx` / `AllowedModelRow.tsx` + specs, `AiSettings.spec.tsx`, `ProviderTabs.spec.tsx`.
- Tests (RTL, follow `essential-test-patterns`): the panel for `openai-compatible` renders the
  base URL field and the attestation switch, and does not render azure fields; the warning text
  matches the reason (`missing-base-url`, `tool-calling-unconfirmed`); an invalid URL shows the
  inline error and blocks submit; env-only disables connection fields; the model row is free
  text with the provider-specific label; the "Test model" button is disabled when connection
  fields are dirty and shows the probe result.

**T1.9 i18n.** Add keys to the five locales. Run the repo's i18n checks if any exist.

**T1.10 Swagger/OpenAPI.** Update enums and schemas; regenerate OpenAPI artifacts if the
`app-commands` skill documents a generation step (check `apps/app/.claude/skills/app-commands/SKILL.md`).

**T1.11 Docs.** Add an env example to `.kiro/specs/ai-provider-multi-vendor/docs/env-configuration-examples.md`
through the amend-spec port-back, plus admin help text. Document server prerequisites (S0.1
table): Ollama context length, vLLM tool parser flags, llama.cpp `--jinja`.

**T1.12 Amend-spec port-back + self-delete** (final task, per `spec-lifecycle.md`).

### 5.11 Manual verification (Objective 1)

Run after T1.1–T1.9, against a real local server (Ollama in a sibling container, reachable as
`ollama`):

1. Admin → AI settings → "OpenAI-compatible" tab: enable, set base URL
   `http://ollama:11434/v1`, leave the key blank, do **not** tick the attestation → save. Expect
   a warning (tool-calling unconfirmed) and the tab dot showing "unavailable". The chat model
   selector must not list the model.
2. Tick the attestation, add model `qwen3:14b` (or whichever S0.1 showed working), set it as
   default, save. "Test model" → reachable, tool call observed (or "not observed" with the
   Ollama note).
3. Chat: ask a wiki question. Check server logs: "Stream finished" with `stepCount > 1` and
   tool calls; no 401; no `OPENAI_API_KEY` usage. To prove the placeholder is used, run
   the app with a dummy `OPENAI_API_KEY=sk-should-not-be-sent`, point the base URL at
   `nc -l 9999` or a request-logging proxy, and confirm the Authorization header carries the
   placeholder, not the env value.
4. Enter `http://user:secret@ollama:11434/v1` → the form rejects it; a direct API PUT → 400.
5. Env-only mode (`AI_USES_ONLY_ENV_VARS_FOR_SOME_OPTIONS=true` plus the §5.4 env example):
   the connection fields are read-only and chat works.
6. Two app instances (if available): save on A, chat on B uses the new config without a restart
   (s2s `configUpdated`).
7. Suggest-path (the editor's "suggest save path" feature) with the local model as the effective
   default: it either works or falls back to memo-only suggestions without a 500 (its
   structured-output run can fail on small models; that is an accepted fallback, record it).

### 5.12 Optional larger scope: N named OpenAI-compatible endpoints

Only if one gateway endpoint is not enough. Sketch, so the cost is visible:

- Config: `ai:customEndpoints` (non-secret array of `{ id, label, baseURL, toolCallingConfirmed, enabled }`)
  and `ai:customEndpointApiKeys` (secret Record by id). `id` is `/^[a-z0-9-]{1,32}$/`, immutable.
- Provider identity: `AiProvider` stays the static union; add a separate `ProviderRef =
  AiProvider | \`custom:${id}\``, used by `AllowedModel.provider`, `ModelKey`, availability,
  resolver dispatch, the chat model list, and the admin UI. `parseModelKey` needs the
  endpoint registry (no longer pure/static), so client code gets the registry from the admin
  GET / chat models API.
- UI: "Add endpoint" / remove / rename-label; tabs become dynamic.
- Everything keyed by `Record<AiProvider, …>` (form values, PUT completeness, swagger,
  `mapProviders`) needs a second dimension.
- Effort estimate: roughly 3–5× the single-slot work; touches every file listed in §2.6's provider table.

---

## 6. Objective 2: make the chat agent research more (small, enforceable code change)

### 6.1 Why local models under-research or "exit early": failure taxonomy

Use these IDs in logs, eval notes and bug reports. Each row names the observable signal
(so it can be detected from the trace in §6.5.8) and the lever that addresses it.

| ID | Failure | Observable signal | Primary lever | Model-dependent? |
|---|---|---|---|---|
| F1 | Answers a wiki question with **no tool call** (from parametric memory or pure hallucination) | `finishReason=stop`, 0 tool calls | Seed retrieval (§6.5.4) guarantees evidence in context; step-0 `toolChoice:'required'` where honored | Partly (models may still ignore the seed) |
| F2 | Searches once and answers **from snippets** without reading any page | ≥1 search, 0 page reads | Tool-result guidance + sufficiency rule; `enforced` mode keeps page reading open and nudges | Yes (guidance can be ignored) |
| F3 | **Repeats** near-identical queries until `maxSteps` and ends on a tool step (no final text) | `finishReason=tool-calls`, `stepCount == maxSteps`, repeated queries | Novelty tracking; tools removed when stale or on the last step (`toolChoice:'none'`) guarantees a final answer | **No (enforced in code)** |
| F4 | **Context overflow** (Ollama small default `num_ctx`) → system prompt / tool schemas truncated → the model forgets tools and answers badly | High `inputTokens` close to the server limit; behavior degrades after the first page read | Size knobs (page-read line cap, search hit cap, memory window) + server config (`OLLAMA_CONTEXT_LENGTH`) | Server config |
| F5 | Tool call **emitted as text** (vLLM without `--tool-call-parser`, llama.cpp without `--jinja`, weak models) | `finishReason=stop`, text contains `<tool_call>` / `{"name": …}` | Diagnostic flag (§6.5.8) + server setup docs; the probe in §5.6 catches it at setup time | Server/model |
| F6 | **Empty final message** after tool results | `isFinal`, `text.trim()===''` | Empty-answer guard (one retry with tools removed) | **No (enforced)** |
| F7 | Reads a whole long page (`limit: 500`) → context blow-up → F4 | Large tool outputs | Clamp page-read `limit` in the chat wrapper | **No (enforced)** |
| F8 | Asks the user a clarifying question instead of searching | 0 tool calls, question-shaped answer | Seed retrieval (candidates visible) + instructions | Yes |
| F9 | Gives up after **0 hits** instead of retrying in the other language / with a synonym / slug | 1 search with 0 hits, then answer | Guidance on zero hits; budget left; instructions | Yes |

Only F3, F6 and F7 can be fully guaranteed in code with the installed stack (F3/F6 subject to
spikes S0.3.8 and S0.3.10 per provider; see R13). F1 is guaranteed
to have *evidence available* (seed), but not guaranteed to be *used*. The rest is steering. §6.8
states this explicitly.

### 6.2 Current behavior (baseline)

- `growiAgent` (`growi-agent.ts`) has instructions telling the model to search first and read
  candidates (outline → section). There is no minimum research, no novelty rule, and no stop rule.
- `post-message.ts:98-111` calls `growiAgent.stream(messages, { requestContext, maxSteps: 10, memory, providerOptions })`.
  `maxSteps: 10` is only a ceiling. Nothing forces a tool call. If the 10th step is a tool call,
  the stream ends without a final answer (the UI then shows `IncompleteResponseNotice` for a
  non-`stop` finish reason).
- Search returns up to 10 hits by default (max 20) with snippets. Page reads return an outline
  first, then up to 200 lines by default (max 500).
- Memory replays the last 30 messages (`memory/index.ts:31`). For a small local context window
  this alone can be several thousand tokens.

### 6.3 General chat agent vs `suggestPathAgent` (keep them separate)

| | General chat (`growiAgent`) | `suggestPathAgent` |
|---|---|---|
| Entry | `POST /_api/v3/mastra/message` → `post-message.ts` → `agent.stream()` | suggest-path API → `agentic-engine.ts` → `agent.generate()` with `structuredOutput` |
| Model | Per-request (user-selected, allow-list-rounded) | Effective default only (`resolveMastraModel()` with no key) |
| Tools | `fullTextSearchTool`, `getPageContentTool` | `fullTextSearch` = `limitedSearchTool` (budgeted wrapper), `getPageContent`, `listChildren` (separately budgeted) |
| Budgets | None (only `maxSteps: 10`) | `aiTools:suggestPathAgenticSearchLimit` (5), `…ChildListingLimit` (5), `…TimeoutMs` (60 s); `maxSteps` computed from them |
| Research directive | "search first, read candidates" | Multiple search angles; **reserve page reads for 1–2 candidates**; finalize on `limit_exceeded` |
| Output | Streamed Markdown to the UI, persisted to memory | Structured JSON, not persisted |
| Memory | Yes (last 30 messages) | No |

Rules for this objective:

1. **Do not modify** the shared `fullTextSearchTool` / `getPageContentTool` modules' contracts,
   `limitedSearchTool`, suggest-path instructions, or `agentic-engine.ts`. The suggest-path
   agent registers `getPageContentTool` directly, so any behavior change inside that shared
   tool would leak into suggest-path's budget accounting.
2. All chat research policy lives in **chat-only** wrappers plus `post-message.ts` stream options.
3. The chat tool **registration keys** stay `fullTextSearchTool` and `getPageContentTool`. The
   client derives `tool-<key>` part types from them (`interfaces/chat-tools.ts`), and
   `PageSources` lists pages from `tool-getPageContentTool` outputs.

### 6.4 What is enforceable, and where (installed Mastra 1.41.0, verified in §2.3/§2.4)

| Lever | How it is enforced | OpenAI / Anthropic / Google / Azure | vLLM (with tool parser) | Ollama | Notes |
|---|---|---|---|---|---|
| **Seed retrieval** (server runs the search before the loop and puts candidates in context) | Our code | ✅ | ✅ | ✅ | Guarantees evidence is present; does not guarantee the model reads it |
| **Remove tools** (`toolChoice:'none'` or `activeTools` subset) | Mastra strips tools from the request | ✅ | ✅ | ✅ | Reliable "answer now" / "no more searching" |
| **Force a tool call** (`toolChoice:'required'`) | Provider | ✅ | ✅ (vLLM ≥ the version with `required` support *(verify)*) | ❌ ignored | Best-effort only |
| **Budgets / novelty / clamped page size** | Our wrapper tools | ✅ | ✅ | ✅ | Reliable ceilings; guidance text is steering |
| **Tool-result guidance** ("read a page before answering", "retry in Japanese") | Model reads it | steering | steering | steering | Most effective steering channel for small models (recent, short, in the tool result) |
| **System-message nudges per step** (`prepareStep` → `systemMessages`) | Model reads it | steering | steering (check chat template: some templates allow one system message only) | steering (same template caveat) | Optional (enforced mode) |
| **Re-open loop on empty final answer** (`onIterationComplete` → `continue:true`) | Mastra loop | ✅ | ✅ | ✅ | Safe: nothing visible was streamed |
| **Re-open loop on a *non-empty* premature answer** (`isTaskComplete` / `onIterationComplete`) | Mastra loop | ⚠ | ⚠ | ⚠ | The premature answer is already streamed and persisted, so the user sees two answers. **Not in the default design** (strategy S6, experimental) |
| `stopWhen` | Mastra loop | – | – | – | Ends the run after a tool step with **no answer**. Do not use for research control |

**Conclusion (the smallest enforceable change):** seed retrieval + budgeted chat wrapper tools with
novelty tracking + a `prepareStep` policy that removes tools when research is done or the step
budget is almost spent + an empty-answer guard + derived `maxSteps`. `toolChoice:'required'`
on the first step is an extra lever for providers that honor it.

### 6.5 Design

New module directory:
`apps/app/src/features/mastra/server/services/mastra-modules/research/`

```
research/
├── index.ts                 ← barrel: only what growi-agent.ts and post-message.ts import
├── research-settings.ts     ← reads config → ResearchSettings (thin adapter; the only config access)
├── research-ledger.ts       ← ResearchLedger type + PURE reducers (reserve/record)
├── research-policy.ts       ← PURE: computeChatMaxSteps, decideStepPolicy, guidance builders
├── research-trace.ts        ← PURE: summarizeResearchTrace, looksLikeTextualToolCall
├── seed-context.ts          ← PURE: extractLatestUserText, formatSeedContext
├── seed-research.ts         ← adapter: runs the viewer-scoped search with a timeout (never throws)
├── chat-search-tool.ts      ← Mastra adapter: budgeted, novelty-aware wrapper of fullTextSearchTool
├── chat-page-tool.ts        ← Mastra adapter: budgeted, line-clamped wrapper of getPageContentTool
├── research-hooks.ts        ← Mastra adapters: createResearchPrepareStep, createEmptyAnswerGuard
└── request-context.ts       ← GrowiChatRequestContextShape = MastraRequestContextShape & { researchLedger?; chatSearchRetrieval?; researchPageReadMaxLines? }
```

Plus one extraction in `tools/`:
`tools/search-wiki-for-viewer.ts` exports

```ts
export const runWikiSearch = async (args: {
  readonly user: IUserHasId;
  readonly searchService: SearchService;
  readonly input: { query: string; limit: number; sort: SortAxis; order: SortOrder };
  readonly retrieval?: 'keyword' | 'hybrid';   // default 'keyword'; NEVER read from config here
}): Promise<FullTextSearchToolOutput> => { … }
```

It contains everything `fullTextSearchTool.execute` does today after the requestContext guard:
the `elasticsearch_not_configured` early return, user-group resolution, `searchKeyword`,
`formatSearchResult`, projection to `{ pageId, pagePath, snippet }`, and the never-throw
`try/catch`. The tool and the seed step then share **one** permission-correct search path.
`fullTextSearchTool` becomes a thin adapter (read `user`/`searchService` from context →
`context_error` if missing → `runWikiSearch({ …, retrieval: 'keyword' })`). Its output is
unchanged, and its existing spec must pass unchanged.

**Why `retrieval` is an explicit argument:** `limitedSearchTool` (suggest-path) delegates to
`fullTextSearchTool`. If `runWikiSearch` read `ai:chatSearchRetrieval` from config, suggest-path
would silently switch to hybrid in Phase 3, which contradicts D6. Only the **chat wrapper** and
the **seed step** pass a non-default value. Add a test: `fullTextSearchTool` (and therefore
`limitedSearchTool`) always calls `searchKeyword` with `retrieval` absent or `'keyword'`,
regardless of config.

#### 6.5.1 Settings and budgets

New config keys in `config-definition.ts`. They follow the `aiTools:suggestPath*`
precedent: env-driven, read per request, **not** part of the env-only group.

| Key | Env var | Type / default | Meaning |
|---|---|---|---|
| `ai:chatResearchMode` | `AI_CHAT_RESEARCH_MODE` | `'off' \| 'guided' \| 'enforced'`; ships as **`'off'`** (P5), flipped to **`'guided'`** after the §8 evaluation (P6) | `off` = current behavior exactly: today's instructions (composed per mode, §6.5.7), maxSteps 10, no seed, no hooks, wrappers produce today's output. `guided` = seed + budgets + guidance + last-step answer guarantee + empty-answer guard. `enforced` = guided + step-0 `required` when seed hits exist + per-step system nudges + search closed when stale |
| `ai:chatResearchMaxSearches` | `AI_CHAT_RESEARCH_MAX_SEARCHES` | number, default 4 | Model-initiated searches per turn (the seed does not count) |
| `ai:chatResearchMaxPageReads` | `AI_CHAT_RESEARCH_MAX_PAGE_READS` | number, default 6 | `getPageContentTool` calls per turn |
| `ai:chatResearchSeedSearchLimit` | `AI_CHAT_RESEARCH_SEED_SEARCH_LIMIT` | number, default 5; `0` disables seeding | Hits injected as context |
| `ai:chatResearchPageReadMaxLines` | `AI_CHAT_RESEARCH_PAGE_READ_MAX_LINES` | number, default 200 | Clamp for the page tool's `limit` (use 60–120 for small-context local models) |

`research-settings.ts` reads these through `configManager.getConfig`, clamps them
(searches 1–10, reads 1–15, seed 0–10, lines 20–500), and treats an unknown mode as
the key's default value (`'off'` until P6, then `'guided'`) with a `warnOnce`.

```ts
export type ResearchMode = 'off' | 'guided' | 'enforced';
export type ResearchBudget = { readonly maxSearches: number; readonly maxPageReads: number };
export type ResearchSettings = {
  readonly mode: ResearchMode;
  readonly budget: ResearchBudget;
  readonly seedSearchLimit: number;
  readonly pageReadMaxLines: number;
};
```

`computeChatMaxSteps(settings)` (pure):
- `off` → `10` (unchanged).
- else → `maxSearches + maxPageReads + 2` (one answer step + one empty-answer retry), clamped
  to `[4, 30]`. Default: 4 + 6 + 2 = **12**.

#### 6.5.2 Research ledger (pure reducers, per-request state)

```ts
export type SearchRecord = {
  readonly query: string;
  readonly hitPageIds: readonly string[];
  readonly newPageIds: readonly string[];
  readonly failed: boolean;
};

export type ResearchLedger = {
  readonly budget: ResearchBudget;
  readonly mode: ResearchMode;
  readonly searchesReserved: number;          // incremented BEFORE the await (budget safety under parallel calls)
  readonly searches: readonly SearchRecord[];
  readonly pageReadsReserved: number;
  readonly readPageIds: readonly string[];    // pages returned with content (not outline-only)
  readonly seenPageIds: readonly string[];    // union of seed hits and search hits
  readonly seededPageIds: readonly string[];
  readonly emptyAnswerRetries: number;
  readonly emptyAnswerPending: boolean;
};
```

Reducers (all pure, return a new ledger):
`createResearchLedger(settings, seededPageIds)`, `reserveSearch(ledger)` →
`{ ledger, allowed: boolean }`, `recordSearchResult(ledger, query, hitIds | 'failed')`,
`reservePageRead(ledger)` → `{ ledger, allowed }`, `recordPageRead(ledger, pageId, hadContent)`,
`markEmptyAnswer(ledger)`, `consumeEmptyAnswer(ledger)`.

Derived (pure) helpers:

- `searchesLeft`, `pageReadsLeft`.
- `zeroHitSearches`: searches that returned no hits at all. **A zero-hit search is not
  "stale".** It is the signal to *keep* searching with different words (F9). It never
  contributes to the novelty stop.
- `consecutiveStaleSearches`: consider only searches that returned **≥ 1 hit** (skip zero-hit
  and failed ones), and count how many of the most recent of those had
  `newPageIds.length === 0` (every hit already seen). This is the only input to the "no new
  useful pages" stop.
- `unreadCandidateIds`: `seenPageIds − readPageIds`.
- `contentReads`: `readPageIds.length`.

`isResearchSufficient(ledger)` is an explicit truth table. The first matching row wins:

| # | Candidates seen? | `contentReads` | Other condition | Sufficient? | Why |
|---|---|---|---|---|---|
| 1 | no | 0 | `searchesLeft === 0` | **yes** | Budget spent, nothing found → answer "not found in the wiki" |
| 2 | no | 0 | `searchesLeft > 0` | no | Keep searching with different words (even after several zero-hit searches) |
| 3 | yes | 0 | `pageReadsLeft === 0` | **yes** | Cannot read anything more (R4/R7 also cover this) |
| 4 | yes | 0 | otherwise | no | Snippets are not evidence; read a page first |
| 5 | yes | ≥ 1 | `consecutiveStaleSearches ≥ 1` **or** `searchesLeft === 0` **or** `unreadCandidateIds` empty | **yes** | Read something, and further searching adds nothing new |
| 6 | yes | ≥ 1 | otherwise | no | New candidates are still appearing and budget remains |

Required precedence test cases (T2.3), in addition to one case per row:

- two zero-hit searches, `searchesLeft > 0` → **not** sufficient, search stays open, guidance =
  retry rule (§6.5.3 rule 2), and R8 does **not** fire;
- zero-hit, then hits-all-seen, then zero-hit → `consecutiveStaleSearches === 1` (zero-hit
  searches are skipped, they do not reset or extend the streak);
- `maxSteps − 1` reached while row 2 holds → R2 still forces the answer (the answer then says
  "not found").

**Concurrency note (important for a weak implementor):** Mastra runs parallel tool calls
concurrently (default `toolCallConcurrency` 10 when no approval is involved). Keep every
`ctx.get('researchLedger')` → reducer → `ctx.set('researchLedger', next)` sequence
**synchronous** (no `await` between get and set). Reserve before awaiting the search, and
record after it with a second synchronous get/set. That makes budgets exact under parallel
calls. This is the same guarantee `limitedSearchTool` gets by incrementing before delegating,
expressed immutably (coding-style: no mutation).

#### 6.5.3 Chat-only wrapper tools

`chat-search-tool.ts`:

```ts
export const chatSearchTool = createTool({
  id: 'chat-full-text-search-tool',
  description: fullTextSearchTool.description + ' Each call uses one search from a per-answer budget; results mark pages you have already seen, and a `research` field tells you what to do next.',
  inputSchema: fullTextSearchTool.inputSchema!,       // by reference, like limitedSearchTool
  outputSchema: chatSearchOutputSchema,               // superset, see "Output schema" below
  execute: async (input, context) => {
    const ctx = context.requestContext as TypedRequestContext;
    const user = ctx.get('user');
    const searchService = ctx.get('searchService');
    if (user == null || searchService == null) {
      return { result: 'context_error', reason: 'user or searchService missing in requestContext' };
    }
    // Chat-only channel: post-message sets this key from ai:chatSearchRetrieval (Phase 3).
    const retrieval = ctx.get('chatSearchRetrieval') ?? 'keyword';
    const ledger = ctx.get('researchLedger');
    if (ledger == null) {                                                      // mode 'off' → same output as today
      return runWikiSearch({ user, searchService, input, retrieval });
    }

    const reserved = reserveSearch(ledger);
    ctx.set('researchLedger', reserved.ledger);                                // sync get→set
    if (!reserved.allowed) {
      return { result: 'limit_exceeded', reason: 'search budget used up', research: researchStatus(reserved.ledger) };
    }
    const result = await runWikiSearch({ user, searchService, input, retrieval });
    const after = recordSearchResult(ctx.get('researchLedger')!, input.query,  // sync get→set
      result.result === 'ok' ? result.hits.map((h) => h.pageId) : 'failed');
    ctx.set('researchLedger', after);
    return decorateSearchResult(result, after);  // pure: adds hits[i].seen, newHitCount, research{…}
  },
});
```

The chat wrapper calls `runWikiSearch` **directly**, not `fullTextSearchTool.execute`. That is
the only way to pass `retrieval` without changing the shared tool. `chatSearchRetrieval` is a
chat-only key on `GrowiChatRequestContextShape` (§6.5, `request-context.ts`), which
`post-message.ts` sets. Suggest-path's context never has it.

**Output schema (must not be skipped):** Mastra validates tool output against `outputSchema`
and returns the **parsed** value (`validateToolOutput` returns `validation.value`). Zod objects
strip undeclared keys, so any field not declared in the schema **disappears silently** before
the model sees it. `chatSearchOutputSchema` must therefore declare:
- `hits[].seen: boolean` (optional);
- `newHitCount` (optional);
- the `research` object (optional) on the `ok`, `error` and `limit_exceeded` members;
- `hits[].matchedBy` (optional, Phase 3);
- the `limit_exceeded` member.

The same applies to `chatPageOutputSchema` (`research`, `limit_exceeded`). T2.4 must include a
test that runs a wrapper's output through its own `outputSchema.parse()` and asserts that
`research.guidance` survives.

`research` field shape (both wrappers):

```ts
type ResearchStatus = {
  searchesLeft: number;
  pageReadsLeft: number;
  guidance: string;        // one short English sentence, deterministic, from research-policy.ts
};
```

Guidance rules (pure `buildSearchGuidance(ledger, lastResult)`), first match wins:

1. search budget used up → "Search budget used up. Read the most relevant unread pages with getPageContentTool, then answer."
2. 0 hits → "No results. Search again with different words: a synonym, the other language (Japanese ↔ English), fewer keywords, or an English path-style slug."
3. hits returned but all already seen, and `consecutiveStaleSearches ≥ 2` → "Searches are no longer finding new pages. Stop searching. Read the best candidates, then answer."
4. hits returned but all already seen → "No new pages. Change the angle (other language, synonym, or prefix:/path of a promising area) or read the best candidates."
5. hits exist and no page read yet → "Read the most relevant page(s) with getPageContentTool before answering; snippets are not enough evidence."
6. else → "Read any clearly relevant unread page, or answer if you already have enough evidence."

`chat-page-tool.ts` works the same way around `getPageContentTool`:

- Clamp `input.limit` to `settings.pageReadMaxLines`. Put the value in the ledger or in
  `requestContext` (`researchPageReadMaxLines`), so the wrapper does not read config itself.
- Budget: reserve → `limit_exceeded` + guidance "Page-read budget used up. Answer now from the
  pages you have read."
- On `ok`, record `(pageId, hadContent = page.content != null)`.
- Add `research` with page guidance: after an outline-only response → "This was the outline only.
  Call again with offset set to the most relevant heading's line."; after content with
  `hasMore` → "More content follows; read further only if needed."; else the generic rule 6.
- **Keep every existing output field** (`page.pageId`, `path`, `content`, `outline`, …), because
  `PageSources` reads them.

Registration in `growi-agent.ts`:

```ts
tools: {
  fullTextSearchTool: chatSearchTool,    // key unchanged → client part types unchanged
  getPageContentTool: chatPageTool,
},
```

Client typing: `interfaces/chat-tools.ts` must import the **wrapper** output types
(`ChatSearchToolOutput`, `ChatPageToolOutput`, type-only). Check
`client/components/ChatSidebar/page-sources.ts` and `PageSources.tsx`: they must narrow on
`result === 'ok'` and ignore `limit_exceeded`. Add a spec case for a `limit_exceeded` part.

#### 6.5.4 Seed retrieval (pre-retrieval before the loop)

`seed-context.ts` (pure):

- `extractLatestUserText(messages: UIMessage[]): string`: the last `role === 'user'` message's
  text parts, joined with spaces, whitespace-collapsed, truncated to 256 characters. Returns
  `''` when absent.
- `formatSeedContext(hits): string | null`: `null` for zero hits; otherwise:

  ```
  Automatic wiki search for the user's latest message (candidates only; NOT yet read):
  1. /engineering/deploy/k8s (pageId: 64f0…) — "…rolling update with …"
  2. …
  These are leads, not evidence. Read the relevant ones with getPageContentTool before answering.
  If none look relevant, search again with different words. If the message is not about wiki
  content, ignore this list.
  ```

  Snippets arrive as XSS-filtered HTML with `<em class='highlighted-keyword'>`. Strip tags and
  decode entities for the model. Cap each snippet at 160 characters.

`seed-research.ts` (adapter, **never throws**):

```ts
export const seedResearch = async (args: {
  searchService: SearchService; user: IUserHasId; text: string; limit: number; timeoutMs: number;
}): Promise<{ hits: WikiSearchHit[]; contextMessage: CoreSystemMessage | null }> => { … }
```

- Skips (returns empty) when `limit === 0`, `text.length < 2`, or `!searchService.isElasticsearchEnabled`.
- Calls the extracted `runWikiSearch({ searchService, user, input: { query: text, limit, sort: relationScore, order: desc }, retrieval })`
  (`retrieval` = `settings.chatSearchRetrieval`, the same value the chat wrapper gets), so the
  permission path is identical to the tool (user groups + external groups + `formatSearchResult`
  + `canShowSnippet`).
- Uses `Promise.race` with a timeout (default 3000 ms, constant). On error or timeout it logs
  `warn` with a reason code only, and returns empty.
- The query is the raw user text. `parseQueryString` interprets operators; a leading `-word`
  in natural text would become an exclusion. Accept that (rare), or strip leading `-` from
  tokens in `extractLatestUserText`. Decide in the spec; the plan's default is to strip.

Delivery: pass `context: [contextMessage]` to `growiAgent.stream()` (verify in S0.3.6 that it is
not persisted to memory). If S0.3.6 shows it **is** persisted, deliver it instead through
`prepareStep` at `stepNumber === 0` by appending to `systemMessages` with the marker described
in §6.5.5.

Seeded page IDs go into the ledger (`seenPageIds`, `seededPageIds`), so the first model search
that returns the same pages is correctly marked stale.

#### 6.5.5 Per-step policy (`prepareStep`)

Pure decision function in `research-policy.ts`:

```ts
export type StepPolicyInput = {
  readonly stepNumber: number;       // 0-based, from prepareStep args
  readonly maxSteps: number;
  readonly ledger: ResearchLedger;
};
export type StepPolicy = {
  readonly toolChoice?: 'auto' | 'none' | 'required';
  readonly activeTools?: readonly ChatToolKey[];   // 'fullTextSearchTool' | 'getPageContentTool'
  readonly nudge?: string;                          // only used in 'enforced' mode
};
export const decideStepPolicy = (input: StepPolicyInput): StepPolicy => { … };
```

Rules, evaluated in order (first match wins):

| # | Condition | Policy | Why |
|---|---|---|---|
| R1 | `mode === 'off'` | `{}` | Preserve current behavior |
| R2 | `stepNumber >= maxSteps - 1` | `toolChoice:'none'`, nudge "Answer now with what you have." | **Guarantees a final answer** instead of ending on a tool step (F3) |
| R3 | `ledger.emptyAnswerPending` | `toolChoice:'none'`, nudge "Your previous reply was empty. Write the final answer now." | F6 retry |
| R4 | `searchesLeft === 0 && pageReadsLeft === 0` | `toolChoice:'none'` | Nothing left to do |
| R5 | `isResearchSufficient(ledger)` and at least one search or read happened in this turn | `enforced`: `toolChoice:'none'` + nudge "Research complete. Answer from the pages you read; say clearly if the wiki does not contain the answer." / `guided`: `{}` (guidance only) | Stop when there are no new useful pages |
| R6 | `searchesLeft === 0` | `activeTools:['getPageContentTool']` | Search closed, reading still allowed |
| R7 | `pageReadsLeft === 0` | `toolChoice:'none'` | Nothing left to read; searching again is pointless |
| R8 | `mode === 'enforced' && consecutiveStaleSearches >= 2` (hits-but-nothing-new only; zero-hit searches never count) | `activeTools:['getPageContentTool']`, nudge "Stop searching; read the best candidates." | Novelty stop (F3). Zero-hit streaks keep search open (F9) |
| R9 | `mode === 'enforced' && stepNumber === 0 && seededPageIds.length > 0` | `toolChoice:'required'` | Best-effort "read before answering" (ignored by Ollama) |
| R10 | otherwise | `{}` | Model decides |

Adapter `createResearchPrepareStep({ maxSteps })` in `research-hooks.ts`:

```ts
export const createResearchPrepareStep = (opts: { maxSteps: number }): PrepareStepFunction =>
  (args) => {
    const ctx = args.requestContext as TypedRequestContext | undefined;
    const ledger = ctx?.get('researchLedger');
    if (ledger == null) return undefined;
    const policy = decideStepPolicy({ stepNumber: args.stepNumber, maxSteps: opts.maxSteps, ledger });
    if (ledger.emptyAnswerPending) ctx!.set('researchLedger', consumeEmptyAnswer(ledger));   // sync
    return toPrepareStepResult(policy, args.systemMessages, ledger.mode);   // pure mapping
  };
```

`toPrepareStepResult` (pure):
- Map `toolChoice` / `activeTools`; omit keys that are undefined (do not override the loop's
  defaults).
- Nudges only in `enforced` mode. Remove any previous nudge (system message whose content starts
  with the marker `[growi-research]`) from `systemMessages`, then append the new one. That
  de-duplication is needed because `replaceAllSystemMessages` persists for later steps (S0.3.4).
  In `guided` mode, return no `systemMessages` at all, which avoids chat-template problems with
  multiple system messages on some local models.

#### 6.5.6 Empty-final-answer guard

`createEmptyAnswerGuard(requestContext)` returns an `onIterationComplete` handler:

```ts
(ctx) => {
  const ledger = requestContext.get('researchLedger');
  if (ledger == null || !ctx.isFinal) return undefined;
  if (ctx.text.trim() !== '' || ctx.toolCalls.length > 0) return undefined;
  if (ledger.emptyAnswerRetries >= 1) return undefined;          // one retry only
  requestContext.set('researchLedger', markEmptyAnswer(ledger)); // sync
  return { continue: true };   // no `feedback`: it is dropped on a final step anyway (§2.3); R3 delivers the nudge
}
```

The loop still respects `maxSteps` (verified in the loop code: `continue:true` only reopens if
`accumulatedSteps.length < maxSteps`). `computeChatMaxSteps` reserves one step for this retry.

#### 6.5.7 Instructions update (steering, not enforcement)

Rewrite the wiki part of `growiAgent.instructions` as a numbered protocol. Keep the existing
language, link, operator and sort rules. Proposed text:

```
# Research protocol for questions about the wiki
1. An automatic search may already have listed candidate pages in a system message. They are leads, not evidence.
2. Before answering a wiki question, read at least one relevant page with getPageContentTool. Snippets are not evidence.
3. If results are few or off-topic, search again with DIFFERENT words: a synonym, the other language (Japanese ↔ English),
   fewer keywords, or an English path-style slug (e.g. "audit-log"). Never repeat a query.
4. Each tool result has a `research` field with the remaining budget and a guidance sentence. Follow the guidance.
5. Stop searching when searches stop finding new pages or the budget is used up, then answer.
6. If the wiki does not contain the answer after research, say so plainly. Do not invent wiki content.
7. Call tools through the tool interface. Never write a tool call as text.
8. If the message is not about wiki content (greetings, general writing help), answer directly without tools.
```

Keep the instructions short. Small models follow short, numbered, imperative lists better
than long prose. The suggest-path instructions are **not** touched.

**Compose the instructions per mode.** `off` must be byte-identical to today's behavior (it is
the S0 baseline in §8), and item 4 refers to a `research` field that passthrough mode never
returns. Mastra 1.41's `Agent` accepts `instructions` as a `DynamicArgument`
(`({ requestContext }) => string`; verified in `dist/agent/types.d.ts`), so:

```ts
// growi-agent.ts
instructions: ({ requestContext }) =>
  buildGrowiAgentInstructions(
    (requestContext as RequestContext<GrowiChatRequestContextShape>).get('researchLedger')?.mode ?? 'off',
  ),
```

`buildGrowiAgentInstructions(mode)` is a pure function in `research/` (or next to the agent):
`off` returns today's string **unchanged** (move the current literal into a constant, and snapshot-test
it against the current text); `guided`/`enforced` return today's string with the wiki bullet
replaced by the protocol above. The `growi-agent.spec.ts` stub `Agent` captures the config, so the spec can
call the captured `instructions` function with a fake requestContext.

#### 6.5.8 Diagnostics (makes F1–F9 measurable)

`research-trace.ts` (pure):

- `looksLikeTextualToolCall(text)`: regex heuristics for `<tool_call>`, `"name"\s*:\s*"(fullTextSearch|getPageContent)`,
  a leading `{"tool"` or `{"name"`, and ```` ```json ```` blocks containing `"arguments"`. Tested
  with positive and negative fixtures (normal Markdown answers with JSON code examples must
  **not** match; restrict to the tool names).
- `summarizeResearchTrace({ ledger, steps, finishReason, text })` → counts only:

```ts
{
  researchMode, maxSteps, stepCount, finishReason,
  seededHits, searches, distinctQueries, newHitsPerSearch: number[], zeroHitSearches,
  pageReads, contentReads, staleStreakMax,
  forcedFinalAnswer: boolean,        // R2/R4/R5/R7 applied at least once
  emptyAnswerRetry: number,
  textualToolCallSuspected: boolean,
  toolCallsByName: Record<string, number>,
}
```

Extend the existing `logger.info({...}, 'Stream finished')` in `post-message.ts` with
`research: summarizeResearchTrace(...)` and `modelKey` (provider/model, no secrets). **Do not
log queries or page paths at info level**, since they derive from user content. Log them at
`debug` only. (Suggest-path logs its queries in its trace. The chat path is more
privacy-sensitive because it streams user conversations.)

#### 6.5.9 Context-size knobs for small local models (F4/F7)

1. `ai:chatResearchPageReadMaxLines` (above) clamps page reads.
2. The search wrapper passes `limit: Math.min(input.limit, 10)` in `enforced` mode (smaller
   outputs). Keep the default 10 in `guided`.
3. Memory window: `lastMessages: 30` is set at module load (`memory/index.ts`). Mastra's
   per-call `memory` option may accept `options: { lastMessages }` *(verify in S0.3)*. If it
   does, add `ai:chatMemoryLastMessages` (default 30) and pass it per call; if not, leave it
   (and document).
4. Documentation for operators (admin help + docs): Ollama `OLLAMA_CONTEXT_LENGTH` (for
   example 16384 or more for tool use with page reads), vLLM `--max-model-len`, llama.cpp `-c`.
   **This is the single most common cause of local "early exit"**. Put it in the provider help
   text (§5.9).

### 6.6 Tasks (Objective 2)

**T2.1 Extract `runWikiSearch`** (`tools/search-wiki-for-viewer.ts`).
- Move the logic out of `fullTextSearchTool.execute` unchanged (see §6.5 for the signature).
  The tool keeps its schemas, description, and `context_error` / `elasticsearch_not_configured`
  behavior, and always passes `retrieval: 'keyword'`.
- Tests: `full-text-search-tool.spec.ts` and `.integ.ts` pass **unchanged**. New
  `search-wiki-for-viewer.spec.ts` covers user-group resolution being called with the whole
  user object, `formatSearchResult` receiving the same `userGroups`, non-string-path hits being
  dropped (the tests move with the code), and `retrieval` defaulting to `'keyword'`.
  `limited-search-tool.spec.ts` gets one new case: the delegated search is always keyword,
  even when `ai:chatSearchRetrieval` is `'hybrid'` (mock `configManager` to prove it is never
  consulted).

**T2.2 Settings + config keys.**
- Files: `config-definition.ts` (5 keys, §6.5.1), `research/research-settings.ts` + spec.
- Tests: defaults; clamping; unknown mode → the configured default + warn once.

**T2.3 Ledger + policy (pure).**
- Files: `research-ledger.ts`, `research-policy.ts` + specs.
- Tests: table-driven over R1–R10 (one case per row plus precedence cases, e.g. R2 beats R9 at
  `maxSteps === 1`); reserve/record under simulated interleaving (reserve, reserve, record,
  record); `isResearchSufficient` truth table; guidance rule order; `computeChatMaxSteps`
  clamps.

**T2.4 Wrapper tools.**
- Files: `chat-search-tool.ts`, `chat-page-tool.ts` + specs.
- Tests: use `mock<SearchService>()` and a mocked delegate `execute` (spy on
  `fullTextSearchTool.execute` / `getPageContentTool.execute` via `vi.spyOn`, or inject). Cover:
  no ledger → exact passthrough (deep-equal to the delegate output); budget exhausted →
  `limit_exceeded` **without** calling the delegate; a delegate error still consumes budget;
  novelty marks `seen` correctly across calls; the page tool clamps `limit`; the page tool records
  content vs outline-only reads; every original output field is preserved.

**T2.5 Seed retrieval.**
- Files: `seed-context.ts`, `seed-research.ts` + specs.
- Tests: `extractLatestUserText` (multi-part messages, non-text parts, the last user message
  wins, truncation, leading `-` stripping); `formatSeedContext` (tag stripping, entity decoding,
  snippet cap, `null` on zero hits); `seedResearch` skips when ES is disabled, returns empty on a
  timeout (fake timers) and on a thrown error, and never throws.

**T2.6 Hooks.**
- Files: `research-hooks.ts` + spec.
- Tests: `createResearchPrepareStep` returns `undefined` without a ledger; it maps policy to
  result keys and omits undefined ones; it de-duplicates the nudge marker; the empty-answer
  pending flag is consumed. `createEmptyAnswerGuard` reopens only for final + empty + no tool
  calls + first time.

**T2.7 Wire into the agent and route.**
- `growi-agent.ts`: tools → wrappers (keys unchanged); instructions → §6.5.7.
- `post-message.ts`: read settings; create the ledger; run the seed (after thread creation, before
  `stream`); pass `maxSteps`, `prepareStep`, `onIterationComplete`, `context`; extend the
  "Stream finished" log. Keep the handler thin. If it grows, move the stream-options assembly into
  `research/index.ts` as `buildResearchStreamOptions(settings, requestContext, seed)`.
- Tests: `growi-agent.spec.ts`: tool keys are exactly `['fullTextSearchTool','getPageContentTool']`
  and resolve to the wrappers; the captured `instructions` function returns today's exact
  text for a context without a ledger (and for mode `off`), and the protocol text for `guided`. `post-message-handler.spec.ts`:
  with the `off` mode the stream options equal today's (`maxSteps: 10`, no `prepareStep`, no
  `context`); with `guided` they include derived `maxSteps`, `prepareStep`, `onIterationComplete`,
  and `context` only when seed hits exist; a seed failure does not fail the request.

**T2.8 Client typing.**
- `interfaces/chat-tools.ts` uses the wrapper output types; the `page-sources.ts` spec gets a
  `limit_exceeded` case.

**T2.9 Diagnostics.**
- `research-trace.ts` + spec (fixtures for textual tool calls, including false-positive guards).

**T2.10 Docs / help text** for the modes and the context-size knobs.

### 6.7 What stays model-dependent (state this in the spec)

1. Whether the model **uses** the seed candidates rather than answering from memory (F1/F8).
2. Whether it reads a page before answering when tools are available (F2). `enforced` +
   OpenAI/Anthropic/Google/Azure/vLLM makes the *first* step a tool call, but not *which* tool,
   nor whether the answer is grounded in what was read.
3. Query quality: distinct angles, the other language, slugs (F9).
4. Whether tool calls are emitted as structured calls at all (F5). That is a server/model setup
   property; the probe in §5.6 detects it.
5. Answer faithfulness to the pages read.

Everything else is enforced in code: search/read ceilings, novelty accounting, the guaranteed
final answer (no tool-step ending; per provider, subject to spikes S0.3.8/S0.3.10 and R13), the empty-answer retry, page-size clamping, and the
availability of seed evidence.

---

## 7. Objective 3: Elasticsearch semantic / embedding retrieval (and other native options)

### 7.1 Current state, precisely

- Both mappings declare `body_embedded: { type: 'dense_vector', dims: 768 }` (no explicit
  `index` / `similarity`).
- `prepareBodyForCreate` writes `body_embedded: page.revisionBodyEmbedded`.
- `aggregatePipelineToIndex` never projects `revisionBodyEmbedded`, so the property is always
  `undefined` and ES stores no vector. **No code generates embeddings, and no query reads the
  field.** It is dormant scaffolding. Its only current effect: the mapping (and its dims) is
  fixed in every existing GROWI index.
- Chat search goes `fullTextSearchTool` → `SearchService.searchKeyword` →
  `ElasticsearchDelegator.search` (multi_match over `path.ja/en^2`, `body.ja/en`,
  `comments.ja/en` + operator filters + permission filter + `function_score` on bookmarks +
  highlight).

### 7.2 Constraints that shape the design

1. **Every page view re-indexes the page.** `addSeenUsers` → `syncPageUpdated` →
   `updateOrInsertPageById` → aggregation → bulk `index` (full replace). Bookmark, comment and
   tag events do the same. Consequences:
   - Embedding must **not** be computed inside `updateOrInsertPages`. That would mean one
     embedding call per page view.
   - A vector written only to ES (e.g. by a partial `_update`) is **erased** by the next full
     `index` op from the aggregation. The vector must come *from the aggregation*, which means
     it must be stored in Mongo. This is exactly what the dormant `revisionBodyEmbedded` hook
     anticipates.
2. **Bulk item errors are swallowed** (logged only). A vector with the wrong length rejects the
   *whole document*, so the page silently vanishes from keyword search too. The writer must
   validate the vector length and omit the field when it is invalid.
3. **`dims` is immutable per index.** Model/dims changes require a new index (the existing
   rebuild path creates new indices, so it can apply a new mapping).
4. **Both ES 8 and ES 9 are supported** (`app:elasticsearchVersion` 8 | 9). `semantic_text` is GA
   only on 9.0+.
5. **Japanese content** (kuromoji analyzers are first-class here). English-centric models (ELSER,
   `nomic-embed-text`) are a poor default.
6. **Air-gapped / self-hosted installs are first-class** (the model-catalog specs forbid
   runtime external calls by default). The embedding endpoint must be admin-configured, and
   local (Ollama/vLLM) must work.
7. **ACL:** the ES `bool.filter` built by `filterPagesByViewer` is the **only** listing gate.
   `formatSearchResult` re-loads pages by `_id $in` with no viewer check. Every retrieval path
   must carry the identical filter.
8. **Exact identifiers must keep working:** keyword search stays; semantic only adds recall.

### 7.3 Options evaluated (support, deployment, licensing)

Licensing statements are from Elastic's current subscription page and docs (fetched
2026-09-24). **Re-verify for the exact ES version and license of each deployment.**

| # | Option | ES 8 | ES 9 | Deployment needs | Japanese fit | Verdict |
|---|---|---|---|---|---|---|
| O1 | **App-side dense embeddings** → `dense_vector` + kNN | ✅ (top-level `knn` with `filter`; minimum minor from S0.5) | ✅ | An embedding endpoint (OpenAI, Azure, or OpenAI-compatible such as Ollama `/v1/embeddings`); app worker + Mongo store | Depends on the chosen model (bge-m3, multilingual-e5, embeddinggemma are multilingual) | **Default (D7)**. Works on both versions, any license listed for vector search, local-first |
| O2 | `semantic_text` field + inference endpoint (`openai` service with custom `url`, or `elasticsearch` service with E5/ELSER) | preview on late 8.x only | ✅ GA 9.0+ | ES must reach the embedding server; secret stored in ES; ES re-infers on ingest *(verify whether unchanged text is re-embedded on a full re-index; with page-view re-indexing this could be very costly)*; docs mention "appropriate license" | Depends on model | Optional (§7.10), ES 9 only |
| O3 | ELSER (sparse, `sparse_vector` / `semantic_text` default on some versions) | ✅ (ML node) | ✅ (ML node) | Dedicated ML node memory; model download (manual in air-gapped installs) | ELSER v2 is English-oriented *(verify for any multilingual ELSER variant)* | Not recommended as the default for this product |
| O4 | E5 multilingual deployed inside ES (ML node, `elasticsearch` inference service) | ✅ (ML node) | ✅ | ML node; trained model deployment | Good (multilingual) | Optional for ES-centric operators who don't want an external endpoint |
| O5 | **Hybrid by app-side RRF** (lexical request + kNN request, fused in Node) | ✅ | ✅ | none | – | **Default (D9)**. No version/licensing dependency; keeps `sort`, `function_score`, highlight in the lexical request |
| O6 | Hybrid by `retriever: { rrf \| linear }` | version-dependent *(verify minimum 8.x minor for retrievers/RRF)* | ✅ | none | – | Optional. Forbids top-level `sort`, so it cannot serve the chat tool's `updatedAt`/`createdAt` sorts |
| O7 | Hybrid by `knn` + `query` in one request (scores summed with boosts) | ✅ | ✅ | none | – | Not recommended: BM25 (unbounded) + cosine ([0,1]) score scales need per-wiki tuning |
| O8 | Reranking (`text_similarity_reranker` retriever with an inference reranker, or app-side LLM rerank) | varies | ✅ | reranker endpoint | model-dependent | Later |

"Other native search methods" that need **no** embeddings. These are cheap and complementary,
and are worth evaluating in the §8 harness:

| Method | What it gives | Cost | Recommendation |
|---|---|---|---|
| `more_like_this` query (same permission filter) | "Pages similar to this page" from term statistics; good follow-up after the agent reads a page | Small: a new chat tool or a mode of the search wrapper | Good optional tool (`relatedPagesTool`) for Obj 2; no model needed; works on ES 8/9 |
| Synonym graph filter (search-time) | Maps domain synonyms (e.g. JP↔EN product terms) | Index settings change + a maintained synonyms set (ES 8.10+ has a synonyms API *(verify)*) | Only if the evaluation shows synonym misses dominate |
| `fuzziness: 'AUTO'` on `path.en`/`body.en` | Typo tolerance for Latin text | Query change only | Low value for kuromoji text; test on EN typos |
| `multi_match` `cross_fields` / `best_fields` alternatives | Different term-to-field scoring | Query change only | Tune only with evidence |
| Agent-side query expansion (other language, slug) | Already prompted; Obj 2 guidance strengthens it | none | Keep (Obj 2) |

**Semantic retrieval raises candidate recall. It cannot make the agent call tools.** It helps
the agent only through the search tool and the seed retrieval (§6.5.4), both of which will use
the hybrid path when enabled.

### 7.4 Recommended minimal design

#### 7.4.1 Embedding model configuration (reuses Objective 1's provider layer)

New config key (not secret; the key comes from `ai:providerApiKeys`):

| Key | Env | Shape | Default |
|---|---|---|---|
| `ai:embeddingModel` | `AI_EMBEDDING_MODEL` | `{ provider: 'openai' \| 'openai-compatible' \| 'azure-openai', modelId: string, dimensions: number }` (JSON) | `null` = semantic retrieval disabled |
| `ai:embeddingMaxInputChars` | `AI_EMBEDDING_MAX_INPUT_CHARS` | number | 2000 (≈ safe for 512-token models with CJK text; raise for 8k-token models) |
| `ai:chatSearchRetrieval` | `AI_CHAT_SEARCH_RETRIEVAL` | `'keyword' \| 'hybrid'` | `'keyword'` |

- Embedding availability is **not** the chat availability rule. The provider needs its
  connection config (openai: key; openai-compatible: baseURL; azure: endpoint + key/Entra), but
  neither `enabled` nor `toolCallingConfirmed` (irrelevant for embeddings). Implement it as a
  small pure function `evaluateEmbeddingAvailability(input)` next to the chat rule, so both stay
  in the client-safe interfaces module.
- Resolver `llm-providers/embedding.ts` (dynamic import, like the chat resolvers):
  - `openai`: `createOpenAI({ apiKey, baseURL: 'https://api.openai.com/v1' }).embedding(modelId)`
    (explicit baseURL, see OD-6).
  - `openai-compatible`: `createOpenAI({ baseURL, apiKey: key ?? placeholder }).embedding(modelId)`.
  - `azure-openai`: `createAzure({...}).embedding(deploymentName)` (Entra path as in chat).
  - For OpenAI `text-embedding-3-*`, pass `providerOptions: { openai: { dimensions } }` so the output
    matches the configured dims. For other models, dims are fixed by the model; the probe below
    detects mismatches.
- `embedMany` / `embed` from `ai` (installed) do batching and retries. Pass `maxRetries: 2` and
  an `abortSignal` timeout.
- An admin "Test embedding" action (reuse the probe route pattern from §5.6): embeds a short
  string, reports the returned vector length vs configured `dimensions`, and reports the latency.

`embeddingVersion` (pure, `semantic-search/embedding-version.ts`):
`${provider}/${modelId}@${dimensions}/r${EMBEDDING_RECIPE_VERSION}`, where
`EMBEDDING_RECIPE_VERSION` is a code constant bumped whenever `buildEmbeddingText` changes.
Every stored vector and the index `_meta` carry it. A mismatch means "stale".

#### 7.4.2 Embedding text recipe (pure)

`buildEmbeddingText({ path, body }, maxChars)`:

- Title = last path segment (decoded). **Not the full path.** Otherwise renaming a parent would
  force re-embedding every descendant; hierarchy recall stays with lexical search.
- Body: strip fenced code blocks longer than N lines (keep short ones), collapse whitespace,
  strip Markdown image/link URLs (keep link text), keep headings.
- Text = `title + '\n\n' + body`, truncated to `maxChars` at a character boundary (never split
  a surrogate pair).
- `contentHash = sha256(embeddingVersion + '\0' + text)` (hex).

Pages longer than `app:elasticsearchMaxBodyLengthToIndex` are indexed with `body: ''` today, so
they are invisible to keyword search. They **will** get a (truncated) vector, a recall
improvement. Record this in the spec as intended behavior.

#### 7.4.3 Vector store (Mongo)

Collection `page_embeddings` (model name e.g. `PageEmbedding`). **Name the collection
explicitly.** Mongoose would otherwise pluralize `PageEmbedding` to `pageembeddings`, and the
aggregation `$lookup` below would silently match nothing. Export one constant
`PAGE_EMBEDDINGS_COLLECTION = 'page_embeddings'` from the model module. Pass it as the schema's
`collection` option, and have `aggregate-to-index.ts` import the same constant for `from:`.

```ts
{
  page: ObjectId,            // indexed; unique together with embeddingVersion
  revision: ObjectId,        // the revision the vector was computed from (observability)
  embeddingVersion: string,
  contentHash: string,
  vector: number[],          // length === dimensions (see storage note)
  updatedAt: Date,
}
// unique index: { page: 1, embeddingVersion: 1 }
```

- Keyed by **page**, not revision, so the aggregation can `$lookup` by page `_id` and an update
  simply upserts (no orphans per revision). Deleted pages are cleaned up on `deleteCompletely`
  and `syncDescendantsDelete`. A periodic sweep also removes rows whose page no longer exists,
  and rows with a non-current `embeddingVersion` after a successful model switch.
- Storage: `number[]` stores BSON doubles (768 × 8 B ≈ 6 KB per page; 100k pages ≈ 600 MB). A
  `Binary` of float32 halves that, but needs decoding in `prepareBodyForCreate` because
  `$project` cannot decode it. **Default `number[]`** (simplest); binary is OD-5.
- Model layer: `.claude/rules/model.md` says Mongoose keeps owning index creation during the
  Prisma migration. `refreshed-model-catalog.ts` is Prisma-first only because that collection has
  **no secondary index**. This collection **needs** a unique compound index. Define it with a
  Mongoose schema (which creates the index), and add a Prisma model + `defineExtension` only if
  the model rule requires it for new collections. Run the `mongoose-to-prisma` skill's guidance
  check, and ask the reviewer if unsure.
- Export/import (G2G transfer, archive export): decide whether `page_embeddings` is included. The
  default is **excluded** (derivable data, can be regenerated by backfill). Check the export
  collection lists under `apps/app/src/server/service/export*` / `import*` and the
  `g2g-transfer-migration-mode` spec.

#### 7.4.4 Index pipeline changes

1. `aggregate-to-index.ts`: add an optional parameter `embeddingVersion?: string`. When set, add
   before `$project`:
   ```ts
   { $lookup: {
       from: PAGE_EMBEDDINGS_COLLECTION,   // the constant exported by the model module
       let: { pageId: '$_id' },
       pipeline: [
         { $match: { $expr: { $and: [ { $eq: ['$page', '$$pageId'] }, { $eq: ['$embeddingVersion', embeddingVersion] } ] } } },
         { $project: { _id: 0, vector: 1 } },
         { $limit: 1 },
       ],
       as: 'embedding',
   } },
   ```
   and project `revisionBodyEmbedded: { $arrayElemAt: ['$embedding.vector', 0] }`. When
   `embeddingVersion` is undefined, the pipeline is **byte-identical to today's**, which the
   integ test asserts.
   - The `$lookup` uses the `{page, embeddingVersion}` unique index (the `$expr` equality on the
     leading field). Verify with `explain` on a realistic dataset (S0-style check).
   - Stale-but-present vectors: the lookup attaches the latest vector for the page regardless of
     revision, so a just-edited page keeps its previous vector for the few seconds until the
     worker re-embeds it. Accept this (better than dropping the page from semantic results).
2. `elasticsearch.ts#updateOrInsertPages`: read the effective `embeddingVersion` (or `undefined`
   when semantic is disabled or the index `_meta` version mismatches, see 7.4.7) once per call and pass it
   to the pipeline.
3. `elasticsearch.ts#prepareBodyForCreate`: set `body_embedded` only when
   `isValidVector(page.revisionBodyEmbedded, this.embeddingDims)` (pure: array, exact length, all
   finite numbers). Otherwise omit the property. **Never send an invalid vector.**
4. Mappings: turn the static `mappings` objects into the existing object plus a pure
   `withEmbeddingMapping(mappings, embedding: { dims, version } | null)`:
   - `null` → return the mappings unchanged (installs without semantic keep today's exact
     mapping, `dims: 768`).
   - otherwise set `body_embedded: { type: 'dense_vector', dims, index: true, similarity: 'cosine' }`
     and **`mappings._meta`** = `{ growi: { embeddingVersion: version } }` (`_meta` lives inside
     `mappings`, not at the top level of the create request).
   `createIndex` applies it for both ES 8 and 9.
5. Rebuild safety: in `ES8/ES9ClientDelegator.reindex`, add
   `script: { source: "ctx._source.remove('body_embedded')", lang: 'painless' }`. The tmp index
   only serves keyword search during a rebuild. Stripping vectors means a dims change can never
   make `_reindex` reject documents (which would drop pages from search mid-rebuild). The main
   index gets vectors back from Mongo through `addAllPages`.

#### 7.4.5 Embedding sync worker (create/update path)

`features/semantic-search/server/services/embedding-sync.ts` (the executor takes the page IDs as input):

- `registerEmbeddingSyncEvents(pageEvent)` listens to `create`, `update`, `revert`
  (revertedPage), and `rename` (the title changes, so re-embed that page only), and to
  `deleteCompletely` / `syncDescendantsDelete` (delete rows). **Read each event's real payload
  shape at its emit site before writing the handler.** `rename` is emitted with a single object
  argument (`page/index.ts:777`, `:1077`), not `(page, user)`. `delete` / `revert` pass
  `(targetPage, otherPage, user)` (`search.ts` `registerUpdateEvent` shows the arities).
  Unit-test each handler with a payload copied from the emit site. Each handler checks
  `isSemanticEnabled()` per event (cheap config read), so enabling needs no restart.
- In-process queue: a `Set<pageId>` with a 3 s debounce, batches of up to
  `EMBED_BATCH_SIZE` (16), and concurrency 1 per instance.
- Per batch:
  1. load pages + current revision bodies;
  2. build texts + hashes;
  3. skip pages whose stored row has the same `contentHash` (no API call);
  4. `embedMany`;
  5. validate each vector's length;
  6. upsert rows;
  7. re-index the batch through `updateOrInsertPages(() => Page.find({ _id: { $in: ids } }))`,
     the single ES write path.
- Failure/retry:
  - API failure → rows are left unchanged; `consecutiveFailures++`;
  - after 3 consecutive failures, open a **circuit** for an exponential backoff
    (30 s → 10 min cap), dropping queued IDs during the open state (the sweep catches them later);
  - log one `warn` per state change (never per page), with a reason code and no text/content.
  - Invalid vector length → treat as a configuration error: open the circuit and `warnOnce`
    "embedding dimensions mismatch (configured X, got Y)".
- Multi-instance: page events fire on the instance that handled the request, so there is no
  duplicate work in the normal flow. Upserts are idempotent anyway.

#### 7.4.6 Backfill, sweep, admin surface

- `embedding-backfill.ts`:
  - cursor over non-deleted pages in `_id` order;
  - for each batch, compute the hash, embed only the missing/stale ones, upsert, and re-index the
    batch;
  - report `{ total, embedded, skipped, failed }`;
  - resumable by nature (hash skip).
- Triggers:
  1. Admin API `POST /_api/v3/search/embeddings/backfill`: admin only, activity-logged,
     single-flight per instance (a module-level promise); a second call while running → 409.
     Progress over the admin socket, mirroring `AddPageProgress` / `FinishAddPage` (new event
     names in `SocketEventName`).
  2. Sweep (node-cron, like `model-catalog-refresh-jobs.ts`): `ai:embeddingSweepCronSchedule`,
     **default `'17 * * * *'` (hourly), active only while semantic retrieval is configured**;
     an empty string opts out. It is both the retry path for failures **and** the only path
     for pages that never emit `create`/`update`: imports, G2G transfers, migrations, and
     possibly duplicates. Check whether `duplicate` (`page/index.ts:1493`, `:1658`) is followed
     by a per-page `create`; if not, subscribe to `duplicate` too, or rely on the sweep.
- Status API `GET /_api/v3/search/embeddings/status`: `{ enabled, embeddingVersion,
  indexEmbeddingVersion, indexNeedsRebuild, pagesTotal, pagesWithCurrentEmbedding, worker: { circuit: 'closed'|'open', lastError?: code } }`.
- UI: a "Semantic search" card in `client/components/Admin/ElasticsearchManagement/`
  (`IndexManagementSection.tsx` area), showing status, a "Generate missing embeddings" button
  with progress, and a warning + "Rebuild index" hint when `indexNeedsRebuild`.
- Cross-instance lock for backfill: OD-7 (default none; idempotent writes make double runs
  wasteful but safe).

#### 7.4.7 Version gate and index/version consistency

At `ElasticsearchDelegator.init()` and after `rebuildIndex`/`normalizeIndices`:

- ES version gate: `getInfo()` gives the node versions. Semantic mode requires every node ≥ the
  minimum version found in S0.5; otherwise `semanticAvailable = false`, with a `warnOnce`.
- Index version: read `mappings._meta.growi.embeddingVersion` of the index behind the alias (add
  `indices.getMapping` to both client delegators). `getMapping({ index: aliasName })` returns an
  object **keyed by the concrete index name** (e.g. `growi`, or `growi-tmp` mid-rebuild), not by
  the alias. Take the single entry and do not look up `response[aliasName]`. If it differs from the configured
  `embeddingVersion` (or is absent), then `indexNeedsRebuild = true`, **semantic queries are
  disabled** (keyword only), and the aggregation stops attaching vectors (avoids a dims
  mismatch). The status API/UI shows "Rebuild index to enable semantic search".
- Cache these flags in the delegator. Re-evaluate on rebuild completion and on the s2s
  `configUpdated` message.

#### 7.4.8 Query path (hybrid via app-side RRF)

API surface: add an optional `retrieval?: 'keyword' | 'hybrid'` to the existing `searchOpts`
object passed through `SearchService.searchKeyword(...)` → `delegator.search(data, user,
userGroups, option)`. **Only `ElasticsearchDelegator` reads it**; `PrivateLegacyPagesDelegator`
ignores it. The global search UI does not pass it (D10).

In `ElasticsearchDelegator.search`:

```
if (option.retrieval !== 'hybrid' || !this.semanticReady() || sort !== relationScore || (offset ?? 0) > 0)
    → today's path, unchanged

lexicalQuery = today's builder (createSearchQuery → criteria → group filter → filterPagesByViewer)
filter       = extractFilterClauses(lexicalQuery)      // PURE, captured BEFORE appendFunctionScore wraps the bool
appendFunctionScore / size / sort / highlight           // unchanged lexical request
text         = semanticQueryText(terms)                // PURE: match + phrase terms joined; '' → keyword only
vector       = await embedQuery(text) with 2500 ms timeout → on failure: keyword only (warn with backoff)
knnRequest   = { index: alias, _source: fields, size: k,
                 knn: { field: 'body_embedded', query_vector: vector, k, num_candidates: max(100, 10k), filter },
                 highlight: highlightWithQuery(text) }  // PURE: highlight_query so kNN-only hits get snippets
[lex, sem]   = await Promise.all([searchKeyword(lexicalQuery), knnSearch(knnRequest)])
return fuseByRrf(lex, sem, { rankConstant: 60, size, rankWindow: max(size, 20) })   // PURE
```

Pure helpers (each with its own spec):

- `extractFilterClauses(query)` → `{ bool: { filter: [...query.bool.filter], must_not: [...query.bool.must_not] } }`.
  It includes the permission `should` block, prefix/tag/author/editor filters,
  `disableUserPages`' `not_prefix`, group filters, and `must_not` exclusions (`-word`,
  `-"phrase"`). It **excludes** scoring `must` clauses. It throws if the permission clause is
  missing, which is a hard guard against building an unfiltered kNN request (see §7.6).
  **Detection:** change `filterPagesByViewer` to push
  `{ bool: { should: grantConditions, _name: VIEWER_PERMISSION_QUERY_NAME } }`, with the constant
  exported from the delegator module. `_name` is valid on any query clause. Its only side effect
  is `matched_queries` on hits, which nothing reads. `extractFilterClauses` then checks for exactly
  one filter clause carrying that `_name`, instead of guessing from structure.
- `semanticQueryText(terms)`: `[...terms.match, ...terms.phrase].join(' ').trim()`.
- `fuseByRrf(a, b, opts)`: standard RRF, `score(d) = Σ 1/(k + rank_i(d))` over the lists containing
  `d`. Deterministic tie-break by best rank, then `_id`. Merge `_highlight` preferring lexical;
  `meta.total = max(a.meta.total, fused.length)`; `meta.took = max`.
- `highlightWithQuery(text)`: the existing highlight settings plus `highlight_query: { multi_match: { query: text, fields: ['body', 'body.ja', 'body.en', 'comments', 'comments.ja', 'comments.en'] } }`.

Client delegators: add `knnSearch(params: estypes.SearchRequest)` to `ES8ClientDelegator` and
`ES9ClientDelegator`. Do **not** reuse `searchKeyword`: its ES9 branch forwards only
`query/sort/highlight` from `body`, and would silently drop `knn`.

Chat integration: `post-message.ts` sets the chat-only requestContext key `chatSearchRetrieval`
from `ai:chatSearchRetrieval`; the chat search wrapper and the seed step pass it to
`runWikiSearch` (T2.1). `fullTextSearchTool` / `limitedSearchTool` (suggest-path) always use
`keyword`.
Optionally add `matchedBy: 'keyword' | 'semantic' | 'both'` per hit to the chat wrapper output. It
tells small models that a semantic-only hit may not contain their literal words. The fusion
result must then carry the provenance.

### 7.5 Tasks (Objective 3)

**T3.1 Config + embedding availability + resolver.** Files: `config-definition.ts`,
`interfaces/embedding-model.ts` (type + pure `evaluateEmbeddingAvailability`),
`llm-providers/embedding.ts` + specs. Tests: availability truth table; the resolver uses
`.embedding()`, the placeholder key when keyless, and an explicit OpenAI baseURL; `dimensions`
providerOption only for OpenAI; lazy import only (`no-eager-provider-imports.spec.ts` still green).

**T3.2 Vector store model.** Mongoose schema + unique index (+ Prisma if required), cleanup
statics. Tests: an integ test (Mongo) for upsert idempotency, the unique constraint, and cleanup
of non-current versions.

**T3.3 Embedding text + version (pure).** Tests: title-only path handling, code-block stripping,
truncation safety (surrogate pairs), hash stability, version string.

**T3.4 Aggregation.** `aggregate-to-index.ts` + `aggregate-to-index.integ.ts`. Tests: without
`embeddingVersion` the pipeline deep-equals the current one (snapshot the array); with it, a page
with a matching row gets `revisionBodyEmbedded`, while a page with a row of another version or no
row gets none. **Insert the embedding rows through the Mongoose model**, never through a
raw collection name. A test that writes to a hard-coded name would pass while production
looks up a different collection.

**T3.5 Writer guard + mapping + reindex strip.** `elasticsearch.ts` (`prepareBodyForCreate`,
`createIndex`), mappings helper, client delegators (`reindex` script, `getMapping`, `knnSearch`).
Tests: `elasticsearch.spec.ts`: invalid vector lengths / NaN → field omitted; a valid vector →
present; `withEmbeddingMapping(null)` returns the original object deep-equal; with config →
`dims`, `index`, `similarity`, `_meta`.

**T3.6 Sync worker.** `embedding-sync.ts` + spec with fake timers: debounce merges bursts; an
unchanged hash skips the API; failure opens the circuit after 3; backoff doubles and caps; a
dims mismatch opens the circuit + warns once; deletes remove rows; re-index is called once per
batch with exactly the batch IDs.

**T3.7 Backfill + status + admin API/UI.** Route specs: admin-only; 409 while running; the status
shape. UI spec for the card (RTL).

**T3.8 Version gate.** Delegator spec: `semanticReady()` is false for ES below the minimum, a
missing `_meta`, a version mismatch, or disabled config; true only when all match.

**T3.9 Hybrid query path.** Pure helpers + delegator spec with mocked client: `retrieval`
absent → exactly one `search` call, with a request identical to today's; hybrid → one lexical +
one `knnSearch` call; the kNN `filter` **deep-equals** the lexical request's filter clauses;
embedding timeout → lexical only; sort `updatedAt` → lexical only; offset > 0 → lexical only.

**T3.10 Chat wiring.** `post-message.ts` sets `chatSearchRetrieval`; the chat wrapper and the seed pass it to `runWikiSearch`; optional `matchedBy` (declared in `chatSearchOutputSchema`). Tests: suggest-path stays keyword; the
option is threaded; output schema.

**T3.11 Live-ES ACL integration tests** (§7.6.3).

**T3.12 Docs:** operator guide (enable → backfill → rebuild → verify), model recommendations,
dims/version semantics, cost notes.

### 7.6 ACL risks and how each is closed

#### 7.6.1 Risks

| Risk | How it could happen | Closure |
|---|---|---|
| A1 Unfiltered kNN | kNN request built without the permission filter | `extractFilterClauses` throws if the permission clause is absent; unit test + live-ES test |
| A2 Post-filter instead of pre-filter | Using `post_filter` or filtering in Node after kNN | Always `knn.filter`. A post-filter is also leak-free, but returns fewer than k; the pre-filter keeps recall **and** safety |
| A3 Filter drift | Lexical filter gains a new clause later (e.g. a new operator), kNN path not updated | Filter is **extracted from the built lexical query**, not rebuilt separately. The drift test asserts deep equality |
| A4 Snippet leak | kNN-only hits get highlights for pages whose snippet must be hidden | Highlights still go through `formatSearchResult` → `canShowSnippet` (unchanged). Test with a GRANT_OWNER page under `hideRestrictedByOwner: false` |
| A5 `formatSearchResult` has no viewer check | Any unfiltered ID list would be rendered | Keep the ES filter as the gate (A1/A3). Optionally add a defense-in-depth viewer check in the chat wrapper (the page tool already enforces `findByIdAndViewer` for bodies) |
| A6 Vectors leak content | `body_embedded` returned in `_source` | `_source` stays the explicit field list (it excludes `body_embedded`); test asserts it |
| A7 Embedding endpoint sees page content | Page bodies are sent to the configured embedding endpoint | This is inherent. Document it: admins choosing a remote endpoint send wiki content there, and local Ollama keeps it on-prem. Same trust model as chat |
| A8 Deleted/trashed pages | Stale vectors for deleted pages | Row cleanup on delete; ES deletes unchanged; the kNN search runs against the same index as lexical, so deleted docs are gone from both |

#### 7.6.2 Unit-level ACL tests

- `extractFilterClauses` over queries built by the real builder methods for: an anonymous user,
  a normal user, a group member, both `hideRestrictedBy*` settings true/false, `disableUserPages`,
  `prefix:`/`-prefix:`/`tag:`/`group:` operators. Assert the permission clause is present and
  identical to the lexical request's.

#### 7.6.3 Live-ES integration tests (`describe.skipIf(!hasElasticsearch)`, following `elasticsearch.integ.ts`)

Fixture: 8 pages in all grant modes (PUBLIC, RESTRICTED(link), SPECIFIED, OWNER(A), OWNER(B),
USER_GROUP(G1), USER_GROUP(G2), a `/user/...` page), all with **the same hand-made vector** (so each
is the nearest neighbor of the query), plus lexical text that matches.

For users {guest, A, B, member of G1} × {hideRestrictedByOwner, hideRestrictedByGroup} ∈
{true,false}² × {`disableUserPages` on/off}: hybrid results' page IDs ⊆ the set that
**keyword** search returns for the same user and settings, **plus** semantic-only matches that
satisfy the same grant rules. Compute the expected sets from the grant rules in the test, not by
calling the code under test.

### 7.7 Migration and re-index implications

| Situation | What happens | Operator action |
|---|---|---|
| Upgrade GROWI, semantic **disabled** (`ai:embeddingModel` null) | Mapping unchanged; pipeline unchanged; no new work | None |
| Enable semantic for the first time | Index `_meta` absent → `indexNeedsRebuild` → keyword only | 1) configure the model, 2) **Rebuild index** (new mapping with configured dims, `index: true`, `_meta`), 3) **Generate embeddings** (backfill). Order 3→2 also works: rebuild attaches the already-stored vectors |
| Change model or dims | New `embeddingVersion` → stale everywhere → keyword only until done | Backfill (new-version rows) → rebuild → the sweep removes old-version rows. For zero-downtime model switches, see OD-8 |
| Change the embedding text recipe (code release bumps `EMBEDDING_RECIPE_VERSION`) | Same as a model change | Release notes must say "re-run backfill + rebuild" |
| ES 8 → 9 upgrade | Rebuild on the new cluster (existing practice); vectors come from Mongo | Rebuild |
| Rebuild while backfill runs | The rebuild's `addAllPages` attaches whatever vectors exist; the worker's per-batch re-index fills the rest | None (both idempotent) |
| Disable semantic | Queries fall back to keyword; the pipeline stops attaching vectors; existing vectors stay in ES until the next rebuild | Optional rebuild; optionally drop `page_embeddings` |
| Very large wiki (100k+ pages) | Backfill time = pages / throughput (e.g. Ollama on GPU ~100–500 texts/s for small embedding models; CPU far slower) *(measure in S0.6)*; storage §7.4.3 | Schedule off-hours; the sweep catches failures |

### 7.8 Failure behavior summary

| Failure | Index path | Query path |
|---|---|---|
| Embedding endpoint down | Worker circuit opens; pages index without new vectors (old vectors stay attached) | Query embedding timeout → keyword only (warn with backoff) |
| Wrong dims configured | Worker detects the length mismatch → circuit open + warnOnce; the writer never sends invalid vectors | `semanticReady()` false if the index `_meta` mismatches |
| ES below the minimum version | – | Semantic disabled; warnOnce |
| kNN request error (e.g. mapping lacks `index: true`) | – | Catch → keyword-only result for this request + warn (never 500 the chat tool) |
| Mongo `page_embeddings` lookup slow | Aggregation slower | – (monitor with `explain`; the index covers it) |

### 7.9 Tests to add or update (Objective 3 summary)

| Spec | New/updated | Type |
|---|---|---|
| `embedding-version.spec.ts`, `embedding-text.spec.ts` | new | unit (pure) |
| `interfaces/embedding-model.spec.ts` | new | unit |
| `llm-providers/embedding.spec.ts` | new | unit (mock `@ai-sdk/openai`, `@ai-sdk/azure`) |
| `page-embedding.integ.ts` | new | integ (Mongo) |
| `aggregate-to-index.integ.ts` | updated | integ (Mongo) |
| `elasticsearch.spec.ts` | updated | unit (writer guard, mapping helper, hybrid branch with mocked client) |
| `extract-filter-clauses.spec.ts`, `fuse-by-rrf.spec.ts`, `semantic-query-text.spec.ts`, `highlight-with-query.spec.ts` | new | unit (pure) |
| `embedding-sync.spec.ts`, `embedding-backfill.spec.ts` | new | unit (fake timers, mocks) |
| `search-embeddings` route specs (backfill/status) | new | unit |
| `elasticsearch.semantic.integ.ts` | new | integ (live ES, skipIf) — ACL matrix §7.6.3 |
| `full-text-search-tool.spec.ts` / `search-wiki-for-viewer.spec.ts` | updated | unit (retrieval option threaded) |
| `search-service.integ.ts` | updated | integ (option passthrough; the nq delegator ignores it) |
| Semantic card UI spec | new | component |

### 7.10 Optional larger scope

1. **Chunk-level vectors** (D11): `nested` field `chunks: [{ vector, startLine, heading }]` with
   nested kNN + `inner_hits`. The search result then carries `bestChunk: { startLine, heading }`,
   and the chat wrapper can tell the model "call getPageContentTool with offset=startLine". That
   saves a whole outline round trip, which is valuable for small models. It needs nested kNN
   support in the minimum ES version *(verify)*, more storage, and a chunker (split by headings,
   cap by chars).
2. **ES-native hybrid** via `retriever.rrf` on ES versions that support it, with the same filter
   applied via the retriever `filter`. Keep app-side RRF for sorted requests.
3. **`semantic_text` on ES 9** with an inference endpoint. ES handles chunking/embedding at
   ingest; it removes the worker and the Mongo store. First verify the re-inference cost under the
   page-view re-index pattern, the network path, secret handling in ES, and licensing.
4. **Reranking** of the fused top-N (ES reranker retriever or an app-side LLM call).
5. **Hybrid in the main search UI**, behind a user toggle, with an explanation of why semantic hits
   may not contain the typed words.
6. **`relatedPagesTool`** (MLT, no ML) for the chat agent: "pages similar to this one", with the
   same permission filter.
7. **E5 multilingual inside ES (O4)** for operators who prefer not to run an embedding server.

---

## 8. Evaluation harness: measure research depth and early exits on local models

The objectives are only "done" when measurements show local models research more and
stop exiting early, without hurting cloud models. This section is the test plan for that. It
needs a running dev server, so it runs **after** implementation. Nothing here was executed
while writing this plan.

### 8.1 Success criteria (proposal: confirm in the Objective 2 spec)

Measured per model over the question set in §8.2, 3 runs each:

| Metric | Target for `guided` on local models | Guard for cloud models |
|---|---|---|
| F3 rate: turns ending on a tool step (`finishReason = tool-calls`) | **0 %** | 0 % |
| F6 rate: empty final answers after the one retry | **0 %** | 0 % |
| F1 rate on answerable wiki questions: 0 model tool calls | ≤ 10 % (Ollama), ≤ 5 % (vLLM with `required`) | no regression |
| Read-before-answer rate: ≥ 1 content read on answerable wiki questions | ≥ 80 % | no regression |
| Expected-page recall@read: an expected page was read | ≥ 70 % keyword, higher with hybrid | no regression |
| "Not found" honesty on negative questions | ≥ 80 % | no regression |
| Median latency vs baseline | ≤ 2× | ≤ 1.5× |
| Textual tool-call leaks (F5) | reported, not a target (setup issue) | 0 |

### 8.2 Evaluation wiki and question set

Create a dedicated eval dataset in the dev database (never production). It is a small script,
not committed product code. Put it under `tmp/eval/`, or commit it under `apps/app/bin/dev/` if
the team wants it reusable. It creates ~60 pages through the public API (`POST /_api/v3/page`),
with a mix of:

- Japanese and English pages; some pages whose path slug is English while the body is Japanese
  (e.g. `/tech/audit-log` containing 「監査ログ」);
- long pages (1,000+ lines) where the answer sits in a deep section (tests outline → offset);
- near-duplicate pages (tests novelty and "stop when no new pages");
- pages under `/user/<name>/` (tests `disableUserPages`);
- restricted pages: OWNER, a USER_GROUP the eval user is **not** in (ACL).

Question categories (≥ 5 questions each, ≥ 40 total). Store them as JSONL:
`{ id, category, lang, question, expectedPageIds[], answerNotes, answerable: boolean }`.

| Category | Example | What it probes |
|---|---|---|
| `ID` exact identifier | "What does `AI_TOOLS_SUGGEST_PATH_AGENTIC_TIMEOUT_MS` control?" | Keyword retrieval must still win |
| `SYN` paraphrase / cross-language | JA question about a page written in EN with different wording | F9, hybrid recall |
| `DEEP` answer in a deep section | "What is step 7 of the DR runbook?" | Outline → offset navigation, F7 |
| `MULTI` synthesis across 2–3 pages | "Compare the backup policies of A and B" | Multiple reads, novelty stop |
| `NEG` not in the wiki | "What is our policy on X?" (absent) | Honesty, stop rule, no hallucination |
| `RECENT` recency | "What changed recently in onboarding?" | `sort=updatedAt` path (keyword only) |
| `ACL` restricted content | Ask for content that exists only in a page the user cannot read | Must not reveal its **body or snippet content** (via answer, seed, snippet, or hybrid). Listing a restricted page's **path** is not a leak when `security:list-policy:hideRestrictedBy*` allows listing; that is configured behavior |
| `CHAT` non-wiki | "Translate this sentence to English: …" | Should not force needless research (cost check) |

### 8.3 Strategy matrix

Each strategy is a configuration of the Phase 2/3 switches, so no code branches are needed for
experiments. The one exception is S6, which needs a small experimental flag.

| ID | Config | Purpose |
|---|---|---|
| S0 | `AI_CHAT_RESEARCH_MODE=off` (identical to today by construction, §6.5.7; cross-check once against the pre-change commit) | Baseline |
| S1 | Local, **unshipped** experiment patch: mode `off` but `buildGrowiAgentInstructions` returns the protocol minus item 4 | Isolates the prompt-only effect |
| S2 | `guided`, `AI_CHAT_RESEARCH_SEED_SEARCH_LIMIT=0` | Budgets + guidance + guaranteed answer, **without** seed |
| S3 | `guided` (seed 5) | Adds seed retrieval |
| S4 | `enforced` | Adds step-0 `required`, nudges, stale-close |
| S5 | S3 + `AI_CHAT_RESEARCH_PAGE_READ_MAX_LINES=80` | Small-context tuning |
| S6 | S4 + `isTaskComplete` scorer "≥1 content read for wiki questions" (experimental flag) | Measures the value of post-hoc gating vs its UX cost (double answers) |
| S7 | S3/S4 + `AI_CHAT_SEARCH_RETRIEVAL=hybrid` (after Phase 3) | Semantic recall effect |
| S8 | S7 + chunk-level offsets (only if §7.10.1 is built) | Deep-section navigation |

### 8.4 Model matrix (adjust to available hardware)

| Server | Models (examples) | Notes |
|---|---|---|
| Ollama | `qwen3:8b`, `qwen3:14b`, `llama3.1:8b`, `mistral-nemo`, `gpt-oss:20b` | Set `OLLAMA_CONTEXT_LENGTH` (run the sweep in §8.6); record the Ollama version |
| vLLM | `Qwen/Qwen2.5-7B-Instruct` (hermes parser), `Qwen/Qwen3-14B` | `--enable-auto-tool-choice --tool-call-parser hermes`; tests `required` |
| llama.cpp | a GGUF of one of the above | `--jinja` |
| Cloud control | one OpenAI model + one Anthropic model from the allow-list | Regression guard |

### 8.5 Runner (small script, to be written when evaluating)

`tmp/eval/run-chat-eval.mjs` (Node 22+, no dependencies):

1. Inputs: base URL, a personal access token (with the AI write scope), `modelKey`, strategy
   label, questions JSONL, runs per question.
2. For each question and run: `POST {base}/_api/v3/mastra/message` with header
   `Authorization: Bearer <token>`, and body
   `{ modelKey, messages: [{ id: <uuid>, role: 'user', parts: [{ type: 'text', text: question }] }] }`.
   Omit `threadId`, so every question gets a fresh thread and no memory contamination.
3. Parse the SSE response (`data: {json}` lines). Record per chunk type: tool inputs
   (`tool-input-available`: tool name + args), tool outputs (`tool-output-available`: for the page
   tool, `output.page.pageId`; for search, hit IDs and `research.guidance`), text deltas
   (concatenate), `message-metadata.finishReason`, `error`. Check the chunk type names against
   `ai@6`'s `UIMessageChunk` type.
4. Record wall-clock latency and time-to-first-text.
5. Append a JSONL row:
   `{ qid, category, strategy, modelKey, run, finishReason, text, toolCalls: [{name, args}], readPageIds, searchQueries, latencyMs, ttftMs }`.
6. Separately `grep` the server log for `Stream finished` lines in the run window and join them by
   time order (or by `threadId` if you add it to the log line) to get `research` summaries and
   token usage.

Throttle to one request at a time per local model (GPU contention distorts latency).

### 8.6 Knob sweeps (run on the best 1–2 local models only)

| Knob | Values | Hypothesis |
|---|---|---|
| Ollama context length | 4k, 8k, 16k, 32k | Below ~8–16k, F4 dominates; above, research depth rises |
| `AI_CHAT_RESEARCH_PAGE_READ_MAX_LINES` | 60, 120, 200 | Smaller → more reads fit, less overflow; too small → more offset calls |
| `AI_CHAT_RESEARCH_MAX_SEARCHES` | 2, 4, 6 | Diminishing returns after 4 |
| `AI_CHAT_RESEARCH_SEED_SEARCH_LIMIT` | 0, 3, 5, 8 | Seed helps F1/F8; too many hits add noise and tokens |
| Memory window (if §6.5.9.3 is feasible) | 6, 12, 30 | Smaller improves small models on multi-turn |
| Temperature | server-side only (Ollama Modelfile `PARAMETER temperature`) | Lower → fewer malformed tool calls |

### 8.7 Scoring and analysis

- Automatic from the JSONL + logs: all §8.1 rates, per (model, strategy), with counts. With 3
  runs × 40 questions = 120 turns per cell, report percentages with raw counts (no need for
  fancy statistics; look for large effects).
- Answer correctness (0 = wrong/hallucinated, 1 = partial, 2 = correct and grounded):
  - manual for a 20-question subset;
  - optionally an LLM judge (a strong cloud model) given the question, `answerNotes`, the pages
    read, and the answer. Spot-check the judge against the manual subset.
- ACL category: a **leak** means body text, snippet text, or answer content derived from a page
  the user cannot view (`Page.findByIdAndViewer` would deny it). Any leak is a release blocker,
  regardless of other metrics. Paths listed under the configured list policy are not leaks. Run the
  ACL questions under both `hideRestrictedByOwner/ByGroup` settings.
- Decision rules:
  - The default `ai:chatResearchMode` becomes the lowest mode meeting §8.1 on the target local
    models without breaking cloud guards.
  - If `enforced` nudges degrade some chat templates (e.g. malformed outputs with multiple system
    messages), keep `guided` as the default and document `enforced` per model family.
  - S6 is adopted only if the double-answer UX problem is solved (e.g. by hiding the premature
    answer, which is out of scope here).

### 8.8 Where results go

A results table (model × strategy × metric) and the chosen defaults go into the Objective 2
spec's `research.md` (and the Objective 3 spec's for hybrid). Commit neither raw
transcripts nor eval wiki content; they may contain sensitive text.

---

## 9. Cross-cutting

### 9.1 New and changed config keys (all phases)

| Key | Env var | Default | Secret | Env-only group | Phase |
|---|---|---|---|---|---|
| (existing) `ai:providers` | `AI_PROVIDERS` | null | no | yes | 1 (new `openai-compatible` entry) |
| (existing) `ai:providerApiKeys` | `AI_PROVIDER_API_KEYS` | null | **yes** | yes | 1 (optional new key) |
| (existing) `ai:allowedModels` | `AI_ALLOWED_MODELS` | [] | no | no | 1 (new provider value) |
| `ai:chatResearchMode` | `AI_CHAT_RESEARCH_MODE` | `off` in P5 → `guided` in P6 (after §8) | no | no | 2 |
| `ai:chatResearchMaxSearches` | `AI_CHAT_RESEARCH_MAX_SEARCHES` | 4 | no | no | 2 |
| `ai:chatResearchMaxPageReads` | `AI_CHAT_RESEARCH_MAX_PAGE_READS` | 6 | no | no | 2 |
| `ai:chatResearchSeedSearchLimit` | `AI_CHAT_RESEARCH_SEED_SEARCH_LIMIT` | 5 | no | no | 2 |
| `ai:chatResearchPageReadMaxLines` | `AI_CHAT_RESEARCH_PAGE_READ_MAX_LINES` | 200 | no | no | 2 |
| `ai:chatMemoryLastMessages` (conditional, §6.5.9) | `AI_CHAT_MEMORY_LAST_MESSAGES` | 30 | no | no | 2 |
| `ai:embeddingModel` | `AI_EMBEDDING_MODEL` | null | no | no *(decide: OD-9)* | 3 |
| `ai:embeddingMaxInputChars` | `AI_EMBEDDING_MAX_INPUT_CHARS` | 2000 | no | no | 3 |
| `ai:chatSearchRetrieval` | `AI_CHAT_SEARCH_RETRIEVAL` | `keyword` | no | no | 3 |
| `ai:embeddingSweepCronSchedule` | `AI_EMBEDDING_SWEEP_CRON_SCHEDULE` | `'17 * * * *'` (only runs while semantic is configured; `''` = off) | no | no | 3 |

Config-definition hygiene: add every key to the key list at the top of `config-definition.ts`
(the `CONFIG_KEYS` array next to `'aiTools:suggestPathAgenticSearchLimit'`) **and** to the
definitions object. Keep numeric keys typed `defineConfig<number>`. The loader parses env
strings according to the default's type (the JSON keys use a `null`/object default so the
loader picks its JSON branch, as the existing AI keys document).

### 9.2 New and changed HTTP endpoints

| Method + path | Auth | Phase | Notes |
|---|---|---|---|
| `GET /_api/v3/ai-settings` | admin, READ.ADMIN.AI | 1 | Response gains `providers['openai-compatible']` incl. `openaiCompatibleSettings` |
| `PUT /_api/v3/ai-settings` | admin, WRITE.ADMIN.AI | 1 | `providers` must include `openai-compatible`; new settings validation |
| `GET /_api/v3/ai-settings/available-models?provider=openai-compatible` | admin | 1 | Returns `{ models: [] }` (catalog-less), no code change beyond the enum |
| `POST /_api/v3/ai-settings/test-model` | admin, WRITE.ADMIN.AI, activity | 1 | Probe using saved config only |
| `POST /_api/v3/ai-settings/test-embedding` | admin, WRITE.ADMIN.AI, activity | 3 | Embedding probe (dims check) |
| `POST /_api/v3/search/embeddings/backfill` | admin, activity | 3 | Single-flight; socket progress |
| `GET /_api/v3/search/embeddings/status` | admin | 3 | Coverage + version + circuit state |
| `POST /_api/v3/mastra/message` | login, WRITE.FEATURES.AI | 2 | No request/response contract change; tool output parts gain `research` / `seen` / `limit_exceeded` |

Check the existing admin search routes in `server/routes/apiv3/search.js` for their scope
constants, and use the same ones for the new embeddings routes.

### 9.3 Security checklist (apply to every PR in this plan)

- [ ] No API key value in any log line, error message, API response, or thrown `Error`. Grep the
      diff for `apiKey` inside template strings and logger calls.
- [ ] Keyless custom endpoint sends the **placeholder**, never `process.env.OPENAI_API_KEY`
      (unit test + the manual proxy check in §5.11.3).
- [ ] `baseURL` validated: http/https only, no embedded credentials, no fragment; the config
      accessor re-validates env-provided values; invalid values are never logged verbatim.
- [ ] Probe routes use saved config only; no URL/key accepted from request bodies; admin-only;
      activity-logged; timeouts; sanitized error messages.
- [ ] kNN requests always include the permission filter extracted from the lexical query
      (`extractFilterClauses` throws otherwise); `_source` excludes `body_embedded`.
- [ ] Seed context and tool guidance contain only data the user is already allowed to see (the
      same `runWikiSearch` path, `canShowSnippet` applied).
- [ ] Prompt-injection note: page content read by tools can contain instructions. The research
      policy's enforcement is code-side (budgets, tool removal), so injected text cannot raise
      budgets or re-enable tools. Say so in the spec.
- [ ] Info-level logs contain counts only; queries/paths only at debug.
- [ ] Embedding endpoint choice: admin docs state that page content is sent to the configured
      embedding endpoint.
- [ ] Run the `security-reviewer` agent on the Phase 1 and Phase 3 diffs (repo rule).

### 9.4 Risk register

| ID | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | A custom server implements Chat Completions subtly differently (streaming tool-call deltas, `include_usage`) → broken streams | Medium | Medium | S0.1 matrix; probe route; document supported servers; optional switch to `@ai-sdk/openai-compatible` (D2) |
| R2 | Local model context too small → research makes answers *worse* (more tool output, more truncation) | High | High | Page-read clamp; §8.6 sweep; operator docs; `guided` default clamps |
| R3 | `enforced` nudges (extra system messages) break some chat templates | Medium | Medium | Nudges only in `enforced`; tool-result guidance is the primary channel; eval per model family |
| R4 | Seed retrieval adds latency to every chat turn | Medium | Low | 3 s timeout; the seed is one ES query (~10–100 ms); disable with limit 0 |
| R5 | `context` stream option persisted into memory → the thread grows with seed text | Low | Medium | Spike S0.3.6; fallback to a `prepareStep` system message |
| R6 | Mastra minor update changes `prepareStep` / `onIterationComplete` semantics | Medium | Medium | Pin behavior with the loop-level spike documented; pure-function tests; re-run §8 smoke on Mastra upgrades; note in the spec's Revalidation Triggers |
| R7 | Embedding worker overloads a shared local GPU | Medium | Medium | Concurrency 1, batch 16, circuit breaker, off-hours backfill |
| R8 | Wrong dims silently drop pages from keyword search | Low (after guard) | **High** | Writer length guard; `_meta` version gate; reindex strip script |
| R9 | ACL leak through hybrid | Low | **Critical** | §7.6 closures + live-ES matrix test as a merge gate |
| R10 | Suggest-path breaks when the default model is a small local model (structured output) | Medium | Low | Existing memo-only fallback; document; consider a per-feature model setting later |
| R11 | Storage growth of `page_embeddings` | Medium | Low | Float32 binary option (OD-5); cleanup sweep |
| R12 | Attestation checkbox is ticked without checking → tool-less model selected | Medium | Medium | Probe button; chat error message surfaces provider errors; OD-2 upgrade path |
| R13 | A provider rejects a request whose history has tool calls but which sends no tools (Mastra strips tools on `toolChoice: 'none'`) → R2/R4/R5/R7 cause a 400 instead of an answer | Medium (Anthropic historically) | High | Spike S0.3.10 per provider; fallback via a declared per-provider capability (keep tools, rely on `limit_exceeded`, one extra step) |

### 9.5 Open decisions (defaults already chosen; override only if needed)

| ID | Decision | Default | Changes the plan if overridden |
|---|---|---|---|
| OD-1 | One `openai-compatible` slot vs N named endpoints | One slot (§5.2) | §5.12 becomes the Phase 1 scope (3–5× effort) |
| OD-2 | Tool-calling guarantee strength | Attestation + advisory probe | Persisted verification records required for selection (new store, invalidation on baseURL change) |
| OD-3 | Default `ai:chatResearchMode` after evaluation (ships `off`, P5) | `guided` (P6) | `off` = opt-in rollout; `enforced` = stronger but template risk (R3) |
| OD-4 | Seed query text from raw user message | Yes, with leading `-` stripped | An LLM-free keyword extraction step, or no seed |
| OD-5 | Vector storage format | `number[]` | Float32 `Binary` + decode in `prepareBodyForCreate` |
| OD-6 | Existing `openai` resolver honors `OPENAI_BASE_URL` implicitly | Flag it; **do not change silently** | Pass `baseURL: 'https://api.openai.com/v1'` explicitly (a breaking change for anyone relying on the env var; release note) |
| OD-7 | Cross-instance lock for backfill | None (idempotent) | A Mongo lock document with expiry |
| OD-8 | Zero-downtime embedding model switch | Not supported (keyword only until rebuild) | Two vector fields / blue-green index with dual `_meta` |
| OD-9 | Are `ai:embeddingModel` and the retrieval keys env-only-locked? | No (they are model settings, like `ai:allowedModels`) | Add to the env-only group |

### 9.6 PR slicing and rollout

| PR | Content | Depends on | Default behavior after merge |
|---|---|---|---|
| P1 | Obj 1 T1.1–T1.5, T1.7 (types, rule, resolver, API, form values) | – | New provider exists; hidden until configured |
| P2 | Obj 1 T1.6, T1.8–T1.11 (probe, UI, i18n, swagger, docs) | P1 | Admin can configure local endpoints |
| P3 | Obj 2 T2.1 (extraction only) | – | No behavior change |
| P4 | Obj 2 T2.2–T2.6 (pure modules, wrappers, hooks) with mode default `off` | P3 | No behavior change (the modules are unused until wired) |
| P5 | Obj 2 T2.7–T2.10 wiring, **default `off` first** | P4 | Opt-in via env |
| P6 | Flip the default to `guided` after the §8 evaluation | P5 + eval | Research policy on by default |
| P7 | Obj 3 T3.1–T3.5 (config, store, pipeline, writer guard, mapping) | P1 | No behavior change unless configured |
| P8 | Obj 3 T3.6–T3.8 (worker, backfill, status, version gate, admin UI) | P7 | Operators can build vectors |
| P9 | Obj 3 T3.9–T3.11 (hybrid query, chat wiring, live ACL tests) | P8, P3 | Hybrid opt-in via `AI_CHAT_SEARCH_RETRIEVAL=hybrid` |

Each PR runs `turbo run lint/test/build --filter @growi/app` (AGENTS.md "Before Committing").
Add a changeset only if a published package (`@growi/core`, `@growi/pluginkit`) changes. None
is expected; `escapeStringForMongoRegex` and friends are only consumed.

### 9.7 Definition of done per phase

- **Phase 1:** all T1.x tests green; the §5.11 manual checks pass with Ollama **and** one other
  server; the amend spec is ported back into `ai-provider-multi-vendor` and deleted.
- **Phase 2:** all T2.x tests green; §8 run for S0–S4 on ≥ 2 local models + 1 cloud model; F3 and
  F6 at 0 %; the default mode chosen from the data; results in `research.md`.
- **Phase 3:** all T3.x tests green, **including the live-ES ACL matrix**; backfill + rebuild
  verified on a copy of a realistic dataset; S7 evaluated; operator guide written.

---

## Appendix A: `decideStepPolicy` reference sketch

The sketch shows rule order and shapes. Tests (T2.3) define the contract; adjust names to the final code.

```ts
import type { ResearchLedger } from './research-ledger';
import {
  consecutiveStaleSearches,
  isResearchSufficient,
  pageReadsLeft,
  searchesLeft,
} from './research-ledger';

export type ChatToolKey = 'fullTextSearchTool' | 'getPageContentTool';

export type StepPolicy = {
  readonly toolChoice?: 'auto' | 'none' | 'required';
  readonly activeTools?: readonly ChatToolKey[];
  readonly nudge?: string;
};

const ANSWER_NOW = 'Answer now with the information you have gathered.';
const EMPTY_RETRY = 'Your previous reply was empty. Write the final answer now.';
const RESEARCH_DONE =
  'Research complete. Answer from the pages you read; if the wiki does not contain the answer, say so.';
const STOP_SEARCHING = 'Stop searching; read the most relevant candidates, then answer.';

export const decideStepPolicy = (input: {
  readonly stepNumber: number;
  readonly maxSteps: number;
  readonly ledger: ResearchLedger;
}): StepPolicy => {
  const { stepNumber, maxSteps, ledger } = input;
  if (ledger.mode === 'off') return {};

  if (stepNumber >= maxSteps - 1) return { toolChoice: 'none', nudge: ANSWER_NOW };
  if (ledger.emptyAnswerPending) return { toolChoice: 'none', nudge: EMPTY_RETRY };

  const searches = searchesLeft(ledger);
  const reads = pageReadsLeft(ledger);
  if (searches === 0 && reads === 0) return { toolChoice: 'none', nudge: ANSWER_NOW };

  const acted = ledger.searches.length > 0 || ledger.readPageIds.length > 0;
  if (acted && isResearchSufficient(ledger)) {
    return ledger.mode === 'enforced' ? { toolChoice: 'none', nudge: RESEARCH_DONE } : {};
  }

  if (searches === 0) return { activeTools: ['getPageContentTool'] };
  if (reads === 0) return { toolChoice: 'none', nudge: ANSWER_NOW };

  if (ledger.mode === 'enforced') {
    if (consecutiveStaleSearches(ledger) >= 2) {
      return { activeTools: ['getPageContentTool'], nudge: STOP_SEARCHING };
    }
    if (stepNumber === 0 && ledger.seededPageIds.length > 0) {
      return { toolChoice: 'required' };
    }
  }
  return {};
};
```

Table-driven spec skeleton (follow `essential-test-design`: assert the returned policy, the
observable contract; do not spy on internals):

```ts
describe('decideStepPolicy', () => {
  const base = buildLedger({ mode: 'guided', maxSearches: 4, maxPageReads: 6 });

  it.each([
    ['off mode never intervenes', { ledger: { ...base, mode: 'off' }, stepNumber: 9, maxSteps: 10 }, {}],
    ['last step removes tools', { ledger: base, stepNumber: 11, maxSteps: 12 }, { toolChoice: 'none' }],
    ['empty-answer retry removes tools', { ledger: { ...base, emptyAnswerPending: true }, stepNumber: 3, maxSteps: 12 }, { toolChoice: 'none' }],
    // … one row per rule, plus precedence rows
  ])('%s', (_title, input, expected) => {
    expect(decideStepPolicy(input)).toMatchObject(expected);
  });
});
```

## Appendix B: `fuseByRrf` reference sketch

```ts
type Hit = { readonly _id: string; readonly _score: number; readonly _source: unknown; readonly _highlight?: unknown };

export const fuseByRrf = (
  lexical: readonly Hit[],
  semantic: readonly Hit[],
  opts: { readonly rankConstant: number; readonly size: number },
): (Hit & { readonly matchedBy: 'keyword' | 'semantic' | 'both' })[] => {
  const acc = new Map<string, { hit: Hit; score: number; bestRank: number; inLex: boolean; inSem: boolean }>();
  const add = (hits: readonly Hit[], source: 'lex' | 'sem') => {
    hits.forEach((hit, index) => {
      const rank = index + 1;
      const prev = acc.get(hit._id);
      const contribution = 1 / (opts.rankConstant + rank);
      acc.set(hit._id, {
        hit: prev?.hit._highlight != null ? prev.hit : hit,   // prefer a hit that carries highlights
        score: (prev?.score ?? 0) + contribution,
        bestRank: Math.min(prev?.bestRank ?? rank, rank),
        inLex: (prev?.inLex ?? false) || source === 'lex',
        inSem: (prev?.inSem ?? false) || source === 'sem',
      });
    });
  };
  add(lexical, 'lex');
  add(semantic, 'sem');
  return [...acc.values()]
    .sort((a, b) => b.score - a.score || a.bestRank - b.bestRank || a.hit._id.localeCompare(b.hit._id))
    .slice(0, opts.size)
    .map(({ hit, score, inLex, inSem }) => ({
      ...hit,
      _score: score,
      matchedBy: inLex && inSem ? 'both' : inLex ? 'keyword' : 'semantic',
    }));
};
```

Spec cases: disjoint lists interleave by rank; an overlapping doc outranks single-list docs at the
same ranks; deterministic tie-break; `size` truncation; highlight preference; empty inputs.

## Appendix C: local endpoint setup notes for development

Devcontainer: create `.devcontainer/compose.extend.yml` from `compose.extend.template.yml` and
add a sidecar (GPU passthrough depends on the host). `compose.extend.yml` is git-ignored (`.gitignore:55`):

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    environment:
      - OLLAMA_CONTEXT_LENGTH=16384      # see §6.5.9 / §8.6
    volumes:
      - ollama-data:/root/.ollama
    # deploy.resources.reservations.devices for NVIDIA GPUs, if available
volumes:
  ollama-data:
```

Then, from the app container: `http://ollama:11434/v1` as the base URL. Pull models inside the
sidecar (`ollama pull qwen3:14b`, `ollama pull bge-m3`).

vLLM (separate host or container):

```bash
vllm serve Qwen/Qwen2.5-7B-Instruct --enable-auto-tool-choice --tool-call-parser hermes --max-model-len 32768
```

llama.cpp:

```bash
llama-server -m model.gguf --jinja -c 16384 --port 8080
```

## Appendix D: file index (what each phase touches)

Phase 1 (Objective 1):
```
apps/app/src/features/mastra/interfaces/ai-provider.ts                 (+spec)
apps/app/src/features/mastra/interfaces/openai-compatible-config.ts    (new)
apps/app/src/features/mastra/interfaces/provider-settings.ts
apps/app/src/features/mastra/interfaces/ai-settings.ts
apps/app/src/features/mastra/interfaces/provider-availability-rule.ts  (+spec)
apps/app/src/features/mastra/utils/endpoint-url.ts                     (new, +spec)
apps/app/src/features/mastra/server/services/ai-sdk-modules/llm-providers/config.ts                 (+spec)
apps/app/src/features/mastra/server/services/ai-sdk-modules/llm-providers/provider-availability.ts  (+spec)
apps/app/src/features/mastra/server/services/ai-sdk-modules/llm-providers/openai-compatible.ts      (new, +spec)
apps/app/src/features/mastra/server/services/ai-sdk-modules/llm-providers/index.ts
apps/app/src/features/mastra/server/services/ai-sdk-modules/probe-model.ts                          (new, +spec)
apps/app/src/features/mastra/server/routes/admin-ai-settings/{put,get}-ai-settings.ts               (+specs)
apps/app/src/features/mastra/server/routes/admin-ai-settings/post-test-model.ts                     (new, +spec)
apps/app/src/features/mastra/server/routes/admin-ai-settings/index.ts
apps/app/src/features/mastra/server/routes/admin-ai-settings/get-available-models.ts               (swagger enum)
apps/app/src/features/mastra/client/admin/ai-settings-form-values.ts                               (+spec)
apps/app/src/features/mastra/client/admin/OpenaiCompatibleSettings.tsx                             (new, +spec)
apps/app/src/features/mastra/client/admin/ProviderPanel.tsx                                        (+spec)
apps/app/src/features/mastra/client/admin/AllowedModelsField.tsx, AllowedModelRow.tsx              (+specs)
apps/app/src/features/mastra/client/admin/provider-options-namespace.ts                            (+spec)
apps/app/public/static/locales/{en_US,fr_FR,ja_JP,ko_KR,zh_CN}/admin.json
```

Phase 2 (Objective 2):
```
apps/app/src/server/service/config-manager/config-definition.ts
apps/app/src/features/mastra/server/services/mastra-modules/tools/search-wiki-for-viewer.ts        (new, +spec)
apps/app/src/features/mastra/server/services/mastra-modules/tools/full-text-search-tool.ts          (thin adapter)
apps/app/src/features/mastra/server/services/mastra-modules/research/*                              (new, +specs)
apps/app/src/features/mastra/server/services/mastra-modules/agents/growi-agent.ts                   (+spec)
apps/app/src/features/mastra/server/routes/post-message.ts                                          (+handler spec)
apps/app/src/features/mastra/interfaces/chat-tools.ts
apps/app/src/features/mastra/client/components/ChatSidebar/page-sources.ts                          (+spec)
```

Phase 3 (Objective 3):
```
apps/app/src/server/service/config-manager/config-definition.ts
apps/app/src/features/mastra/interfaces/embedding-model.ts                                          (new, +spec)
apps/app/src/features/mastra/server/services/ai-sdk-modules/llm-providers/embedding.ts              (new, +spec)
apps/app/src/features/semantic-search/server/models/page-embedding.ts                               (new, +integ)
apps/app/src/features/semantic-search/server/services/{embedding-text,embedding-version,embedding-sync,embedding-backfill}.ts (new, +specs)
apps/app/src/features/semantic-search/server/routes/*                                               (new, +specs)
apps/app/src/server/service/search-delegator/aggregate-to-index.ts                                  (+integ)
apps/app/src/server/service/search-delegator/elasticsearch.ts                                       (+spec, +semantic integ)
apps/app/src/server/service/search-delegator/mappings/{mappings-es8,mappings-es9}.ts + mapping helper
apps/app/src/server/service/search-delegator/elasticsearch-client-delegator/{es8,es9}-client-delegator.ts
apps/app/src/server/service/search-delegator/{extract-filter-clauses,fuse-by-rrf,semantic-query-text}.ts (new, +specs)
apps/app/src/server/service/search.ts                                                               (option passthrough)
apps/app/src/server/crowi/*  (register the embedding sync events at startup — locate where SearchService is created)
apps/app/src/client/components/Admin/ElasticsearchManagement/*                                      (semantic card)
apps/app/src/interfaces/websocket.ts (SocketEventName additions)
```
