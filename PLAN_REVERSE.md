# PLAN_REVERSE.md

> **What this file is:** A complete, ordered plan for regenerating the GBrain
> repository from scratch. Execute this plan using `CONTEXT_REVERSE.md` as the
> variable layer. To build a different domain brain, change `CONTEXT_REVERSE.md`
> and re-execute this plan unchanged.
>
> **How to use:** An AI agent (Claude Code, Codex, Cursor, OpenClaw) should read
> `CONTEXT_REVERSE.md` first, internalize every section, then execute the steps
> below in order. Each step references the relevant `CONTEXT_REVERSE.md` section
> in brackets.

---

## Prerequisites

Before generating any code, the agent must verify:

- [ ] Bun ≥ 1.3.10 is installed (`bun --version`)
- [ ] The target directory is empty (or confirmed for overwrite)
- [ ] All values in `CONTEXT_REVERSE.md` §1–§3 are resolved (project name, version, runtime)
- [ ] At least one AI provider API key is available (§6)

---

## Phase 0 — Scaffold & Configuration

### 0.1 — Root files [§1, §3]

Create the following root-level files:

**`package.json`**
- `name`, `version`, `description`, `license` from §1
- `"type": "module"`, `"main": "src/core/index.ts"`, `"bin": {"<name>": "src/cli.ts"}`
- `"exports"` map: expose `./engine`, `./types`, `./operations`, `./minions`,
  `./engine-factory`, `./pglite-engine`, `./link-extraction`, `./import-file`,
  `./transcription`, `./embedding`, `./config`, `./markdown`, `./backoff`,
  `./search/hybrid`, `./search/expansion`, `./ai/gateway`, `./extract`, `./ingestion`,
  `./ingestion/test-harness`
- `"scripts"`:
  - `dev`: `bun run src/cli.ts`
  - `build`: `bun build --compile --outfile bin/<name> src/cli.ts`
  - `build:all`: compile for darwin-arm64 + linux-x64
  - `build:admin`: build React SPA → run `scripts/build-admin-embedded.ts`
  - `build:llms`: `bun run scripts/build-llms.ts`
  - `test`: `bash scripts/run-unit-parallel.sh`
  - `verify`: `bash scripts/run-verify-parallel.sh`
  - `check:all`: chain all check scripts (see §0.5)
  - `typecheck`: `tsc --noEmit`
  - `postinstall`: auto-run `apply-migrations` or print recovery hint
- `"dependencies"`: Vercel AI SDK family, `@electric-sql/pglite@0.4.3`,
  `@modelcontextprotocol/sdk@1.29.0`, `postgres`, `pgvector`, `express@^5`,
  `express-rate-limit`, `cors`, `cookie-parser`, `zod@^4`, `gray-matter`,
  `marked`, `js-yaml`, `chokidar`, `openai`, `web-tree-sitter`, `tree-sitter-wasms`,
  `@dqbd/tiktoken`, `@aws-sdk/client-s3`, `heic-decode`, `@jsquash/avif`,
  `@jsquash/png`, `exifr`, `eventsource-parser`
- `"devDependencies"`: `typescript`, `@types/bun`, `bun-types`, `@types/express`,
  `@types/cors`, `@types/cookie-parser`, `@types/js-yaml`, `fast-check`
- `"engines"`: `{"bun": ">=1.3.10"}`
- `"openclaw"` compat block: `{"compat": {"pluginApi": ">=2026.4.0"}, "extensions": ["./src/openclaw-context-engine.ts"]}`

**`tsconfig.json`**
```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "types": ["bun-types"],
    "strict": true,
    "skipLibCheck": true,
    "noEmit": true,
    "esModuleInterop": true,
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "baseUrl": ".",
    "paths": {"@/*": ["src/*"]}
  },
  "include": ["src", "test"]
}
```

**`bunfig.toml`**
- `[test]` section: `timeout = 60_000` (PGLite cold-start)
- `preload = ["./test/helpers/legacy-embedding-preload.ts"]` (pin legacy embedding dims for tests)

**`gbrain.yml`** [§5 gbrain.yml block]
- `storage.db_tracked` list from §5
- `storage.db_only` list from §5

**`VERSION`**
- Single line: the version string from §1

**`openclaw.plugin.json`**
- `name`, `version`, `description` from §1
- `family: "bundle-plugin"`
- `configSchema` with `database_url` + `openai_api_key`
- `mcpServers.gbrain.command` → `./bin/<name>`, args `["serve"]`
- `skills` array: every skill dir in §8
- `shared_deps` array: conventions + root markdown skill files
- `excluded_from_install` array: `setup`, `migrate`, `publish`
- `openclaw.compat` + `contracts.contextEngines`

**`.gitignore`**
- `node_modules/`, `bin/`, `.env`, `*.db`, `pglite/`, `admin/dist/` and standard TS artifacts
- Any `db_only` prefixes from §5 (machine-generated content)

**`.gitleaks.toml`**
- Rules for detecting leaked API keys (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`,
  `ZEROENTROPY_API_KEY`, `DATABASE_URL`, bearer tokens)

---

### 0.2 — Source tree skeleton

Create empty directories (no files yet):

```
src/
  commands/
    migrations/
  core/
    ai/
      recipes/
    artifact/
    audit/
    bench/
    brainstorm/
    budget/
    calibration/
    chunkers/
    code-intel/
    conversation-parser/
    cross-modal-eval/
    cycle/
    diarize/
    distribution/
    enrichment/
    entities/
    eval/
    eval-contradictions/
    eval-shared/
    extract/
    facts/
    ingestion/
    minions/
    onboard/
    output/
    progressive-batch/
    remediation/
    resolvers/
    schema-pack/
    search/
  eval/
    code-retrieval/
    longmemeval/
    retrieval-quality/
  mcp/
skills/        (see Phase 5)
templates/
recipes/
tools/
docs/
evals/
admin/
scripts/
test/
  e2e/
  fixtures/
  helpers/
examples/
```

---

## Phase 1 — Core Types & Engine Contract

### 1.1 — `src/core/types.ts` [§5]

Define:
- `PageType = string` (open union, not a closed enum)
- `ALL_PAGE_TYPES` array — seed list from §5 canonical page types
- `assertNever(x: never)` — generic exhaustiveness helper for closed enums
- `Page` interface: `id`, `slug`, `type`, `title`, `compiled_truth`, `timeline`,
  `frontmatter`, `content_hash?`, `emotional_weight?`, `created_at`, `updated_at`,
  `deleted_at?`, `effective_date?`, `embedding_signature?`, `source_id?`
- `SearchResult` interface: extends Page with score fields + all boost attribution
  fields (`base_score`, `backlink_boost`, `salience_boost`, `recency_boost`,
  `exact_match_boost`, `graph_adjacency_boost`, `graph_cross_source_boost`,
  `session_demote_factor`, `reranker_delta`, `evidence`, `create_safety`)
- `HybridSearchMeta`, `PageFilters`, `SearchOpts` types
- `Fact`, `Take`, `TimelineEntry`, `GraphNode`, `EdgeType` types
- `BrainConfig`, `EngineKind` (`'pglite' | 'postgres'`) types

### 1.2 — `src/core/engine.ts` [§4]

Define `BrainEngine` interface (the pluggable contract):
- **Page CRUD**: `getPage`, `putPage`, `deletePage`, `deletePages` (batch),
  `listPages`, `listAllPageRefs`
- **Search**: `searchVector`, `searchKeyword`, `searchHybrid`, `getAdjacencyBoosts`
- **Graph**: `addLinks`, `addLinksBatch`, `traverseGraph`, `traversePaths`
- **Embeddings**: `getEmbedding`, `setEmbedding`, `listStaleChunks`,
  `countStaleChunks`, `sumStaleChunkChars`, `setPageEmbeddingSignature`,
  `invalidateStaleSignatureEmbeddings`, `buildBestPerPagePoolCte`
- **Timeline**: `addTimelineEntries`, `addTimelineEntriesBatch`
- **Facts**: `getFacts`, `putFact`, `deleteFact`, `migrateFactsToCanonical`
- **Takes**: `getTakes`, `putTake`
- **Salience**: `getRecentSalience`, `findAnomalies`
- **Transcripts**: `getRecentTranscripts`
- **Aliases**: `resolveAliases`, `setPageAliases`
- **Misc**: `healthCheck`, `applyMigrations`, `getSchemaVersion`,
  `batchLoadEmotionalInputs`, `setEmotionalWeightBatch`,
  `refreshPageBody`, `findOrphanPages`, `resolveSlugsByPaths`
- `readonly kind: 'postgres' | 'pglite'` discriminator
- `clampSearchLimit(limit, default, cap)` helper
- Export `LinkBatchInput`, `TimelineBatchInput`, `TraverseGraphOpts`, `AdjacencyRow`

### 1.3 — `src/core/engine-constants.ts`

Single module: `export const DELETE_BATCH_SIZE = 500`

### 1.4 — `src/core/errors.ts`

- `OperationError` class with `code: ErrorCode`, `message`, `suggestion?`, `docs?`
- `ErrorCode` open union (named values + `(string & {})` autocomplete hack)

---

## Phase 2 — Database Engines

### 2.1 — `src/core/pglite-schema.ts`

SQL DDL strings for all tables:
`pages` · `content_chunks` · `graph_edges` · `timeline_entries` · `facts`
· `takes` · `minion_jobs` · `query_cache` · `audit_log` · `salience_log`
· `calibration_*` tables · `schema_version` · `sources` · `mounts`
· `spend_log` · `eval_candidates` · `page_aliases`

### 2.2 — `src/core/migrate.ts`

- `applyMigrations(engine)` — runs pending migrations in order
- Migration registry: 96+ versioned SQL deltas (v0.11.0 → v0.42.x)
- Branches on `engine.kind` for engine-specific SQL when needed
- Idempotent: each migration checks schema_version before applying

### 2.3 — `src/core/pglite-engine.ts` [§4]

`PGLiteEngine implements BrainEngine`:
- Initializes `@electric-sql/pglite` with `pgvector` extension
- Uses `pglite-schema.ts` for init DDL
- Implements all `BrainEngine` methods with PGLite SQL
- Lock via `pglite-lock.ts` (advisory locks for concurrent write safety)
- WASM grammars embedded from `src/assets/wasm/grammars/`

### 2.4 — `src/core/postgres-engine.ts` [§4]

`PostgresEngine implements BrainEngine`:
- Uses `postgres` npm package (not `pg`)
- Identical SQL logic as PGLite engine (parity maintained by engine-parity tests)
- Connection pooling via `src/core/connection-manager.ts`

### 2.5 — `src/core/engine-factory.ts`

```typescript
export async function loadEngine(config: GBrainConfig): Promise<BrainEngine> {
  if (config.engine === 'postgres') {
    const { PostgresEngine } = await import('./postgres-engine.ts');
    return new PostgresEngine(config);
  }
  const { PGLiteEngine } = await import('./pglite-engine.ts');
  return new PGLiteEngine(config);
}
```

---

## Phase 3 — Configuration & CLI Plumbing

### 3.1 — `src/core/config.ts` [§3, §6, §7]

`GBrainConfig` type + `loadConfig()`:

Seven-tier resolution chain (highest priority first):
1. Per-call flag (CLI `--brain`, `--source`)
2. Environment variable (`GBRAIN_BRAIN_ID`, `GBRAIN_SOURCE`, `DATABASE_URL`, etc.)
3. Per-source DB key
4. Brain-wide DB key
5. `gbrain.yml` (repo root)
6. `~/.gbrain/config.json`
7. Built-in defaults (`engine: 'pglite'`, `search_mode: 'balanced'`)

Key config keys: `engine`, `database_url`, `embedding_model`, `reranker`,
`search_mode`, `search.graph_signals`, `cache.ttl_seconds`, `brain_id`, `source_id`,
`embedding_signature`, `sync.repo_path`

### 3.2 — `src/version.ts`

```typescript
export const VERSION = Bun.file(new URL('../VERSION', import.meta.url)).text().trim();
```
(Or read `VERSION` at import time.)

### 3.3 — `src/cli.ts`

Entry point. Parses global flags (`--brain`, `--source`, `--explain`, `--json`,
`--quiet`, `--non-interactive`) and dispatches to command modules in
`src/commands/`. Sets `OperationContext.remote = false` on all ops (trusted local
caller). Formats output based on `--json` / `--explain` / human modes.

### 3.4 — `src/core/operations.ts` [§9]

**Contract-first single source of truth** for all 47+ operations:
- Each operation: `name`, `description`, `schema` (Zod), `handler`, `scope`,
  `localOnly?`
- `OperationContext` type: `remote: boolean` (REQUIRED), `auth?`, `sourceId?`,
  `allowedSlugPrefixes?`, `brainId?`
- `sourceScopeOpts(ctx)` helper (v0.34.1.0)
- All operations annotated with scope (`read | write | admin`) and `localOnly`
- `sync_brain`, `file_upload`, `file_list`, `file_url` → `admin + localOnly`
- `synthesize`, `patterns`, `consolidate` → protected (PROTECTED job names)

---

## Phase 4 — AI Gateway & Provider Recipes

### 4.1 — `src/core/ai/types.ts`

- `ModelId`, `ProviderKind`, `EmbeddingResult`, `CompletionOpts`, `StreamChunk`
- `AIGateway` interface

### 4.2 — `src/core/ai/gateway.ts` [§6]

**All AI calls route through here.** Responsibilities:
- Provider selection by model ID prefix
- Retry with exponential backoff (`src/core/backoff.ts`)
- Token budget tracking (`src/core/budget/budget-tracker.ts`)
- Spend logging (`src/core/spend-log.ts`)
- Rate limit error propagation (`ErrorCode: 'rate_limited'`)

### 4.3 — `src/core/ai/recipes/` [§6]

One file per provider. Each exports a factory for that provider's client:
`anthropic.ts` · `openai.ts` · `azure-openai.ts` · `google.ts` · `groq.ts`
· `ollama.ts` · `llama-server.ts` · `llama-server-reranker.ts` · `litellm-proxy.ts`
· `zeroentropyai.ts` · `openrouter.ts` · `together.ts` · `deepseek.ts`
· `minimax.ts` · `zhipu.ts` · `dashscope.ts` · `voyage.ts`

### 4.4 — `src/core/ai/model-resolver.ts` [§6]

Maps model ID strings (e.g. `ze:ze-default`, `anthropic:claude-3-5-haiku-latest`)
to provider recipe + client config. Reads `embedding_model` config key.

### 4.5 — `src/core/embedding.ts` [§6]

- `embedText(text, config)` → `Float32Array`
- `embedBatch(texts, config)` → `Float32Array[]`
- Chunking via `src/core/chunkers/`
- `estimateCostFromChars(chars, model)` for cost preview
- `embed-skip.ts`: list of page types / slug prefixes that skip embedding

---

## Phase 5 — Search Layer

### 5.1 — `src/core/search/hybrid.ts` [§7]

`hybridSearch(query, engine, config, opts)`:
1. Query expansion (Anthropic Haiku, optional) — `src/core/search/expansion.ts`
2. Parallel: `searchVector` + `searchKeyword`
3. RRF fusion
4. `runPostFusionStages`: backlink boost → salience boost → recency boost
   → graph adjacency signals (stage 4)
5. Reranker (ZeroEntropy, conditional on mode)
6. `base_score` stamped before any boost stage mutates `score`
7. Per-stage attribution fields stamped on each `SearchResult`

`hybridSearchCached(...)` — wraps with `query_cache` table TTL.

### 5.2 — `src/core/search/graph-signals.ts`

`applyGraphSignals(results, engine, opts)`:
- `ADJACENCY_BOOST = 1.05` — page linked from 2+ OTHER top-K results
- `CROSS_SOURCE_BOOST = 1.10` — page linked from 2+ different sources
- `SESSION_DEMOTE = 0.95` — 3+ results from same chat session (keep highest at full)
- Floor-ratio gate prevents weak pages boosted past strong ones
- Fail-open: errors log to audit, return input unchanged
- `computeScoreDistribution(results)` for instrumentation
- `pairedBootstrapPValue(deltas, resamples, rng)` for eval gates

### 5.3 — `src/core/search/mode.ts` [§7]

`ModeBundle` type with knobs: `vector`, `keyword`, `reranker`, `graph_signals`,
`expansion`, `tokenBudget`. Three named modes: `conservative`, `balanced`, `tokenmax`.
`KNOBS_HASH_VERSION` bumped on any knob change (cache-key contamination guard).

### 5.4 — `src/core/search/explain-formatter.ts`

`formatResultsExplain(results)` → multi-line breakdown per result.
Reads every boost-stamping field. 4-decimal precision with trailing-zero strip.

### 5.5 — `src/core/search/intent-weights.ts`

`applyExactMatchBoost` — title/alias phrase match boost. Stamps `exact_match_boost`.

---

## Phase 6 — Synthesis (Think Layer)

### 6.1 — `src/commands/think.ts`

`gbrain think "<query>"`:
1. Hybrid search (same as `search`)
2. Compose synthesis prompt with top results as context
3. LLM call (Anthropic Sonnet) → cited prose answer
4. Explicit gap analysis section (what the brain doesn't know)
5. Output: cited markdown prose

### 6.2 — `src/core/trajectory.ts`

`findTrajectory(slug, engine, config)`:
Multi-hop graph + timeline traversal to build a "trajectory" view of an entity:
metrics over time, relationships, events, open items. Used by `think` for
entity-centric queries.

### 6.3 — `src/core/calibration/` [§12]

- `cross-brain.ts` — cross-brain calibration aggregation
- `domain-aggregators.ts` — per-domain Brier score computation
- `gstack-coupling.ts` — coupling to GStack bias patterns
- `nudge.ts` — anti-bias rewrite nudge generation
- `svg-renderer.ts` — server-rendered SVG charts for admin SPA
- `voice-gate.ts` — `gateVoice()` gating function for tone enforcement [§20]
- `templates.ts` — hand-written fallback voice templates [§20]

---

## Phase 7 — Link Extraction & Graph Population

### 7.1 — `src/core/link-extraction.ts`

`extractPageLinks(content, schema)`:
- Parse `[[wiki/people/bob]]` wikilink syntax
- Parse typed-link syntax (`works_at::companies/acme`)
- Returns `{slug, edgeType}[]` with zero LLM calls
- `isAutoLinkEnabled(ctx)` — skip for untrusted remote writes
- `makeResolver(engine)` — resolves unresolved refs to existing page slugs

---

## Phase 8 — Ingestion Pipeline

### 8.1 — `src/core/import-file.ts`

`importFromContent(content, opts, engine, config)`:
- Parse frontmatter (gray-matter)
- Infer page type from active schema pack
- Normalize slug
- Chunking (via `src/core/chunkers/`)
- Embed chunks
- `putPage` to engine
- Auto-link extraction
- Timeline entry extraction
- Alias extraction + `setPageAliases`

### 8.2 — `src/core/ingestion/` [§8 ingestion recipes]

`IngestionSource` contract (versioned interface for third-party skillpacks):
- `id`, `name`, `description`, `ingest(item) → Page`
- Built-in sources: `InboxFolderSource`, `MarkdownFileSource`, `ConversationSource`

### 8.3 — `src/core/chunkers/`

Multiple chunkers for different content types:
- `MarkdownChunker` — semantic heading-aware
- `CodeChunker` — tree-sitter-based (uses WASM grammars)
- `PlainTextChunker` — sliding window
- `ImageChunker` — one chunk per image with caption

---

## Phase 9 — Fact & Takes Extraction

### 9.1 — `src/core/facts/` [§5 entity schema]

- `eligibility.ts` — `isFactsBackstopEligible(page)` checks type + flags
- `phantom-audit.ts` — tracks phantom→canonical redirects
- `facts-fence.ts` — fence format for serializing facts in page content
- `takes-fence.ts` — fence format for takes
- `takes-resolution.ts` — resolution of conflicting takes over time

### 9.2 — `src/commands/extract.ts`

`gbrain extract <type>`:
- Runs LLM extraction over pages of the given type
- Writes facts/takes/timeline entries back to engine
- Uses `extract-takes-from-pages.ts`, `extract-timeline-from-meetings.ts`
- Progress events via `src/core/progress.ts`

---

## Phase 10 — Minions (Job Queue) [§10]

### 10.1 — `src/core/minions/`

- `queue.ts` — BullMQ-shaped Postgres-native job queue
- `index.ts` — public API: `submitJob`, `getJob`, `listJobs`, `cancelJob`
- `handlers/` — one handler per job type:
  - `shell-audit.ts` — shell job sandbox + audit
  - `supervisor-audit.ts` — supervisor job tracking
  - `embed-handler.ts`, `extract-handler.ts`, `enrich-handler.ts`
  - `synthesize-handler.ts`, `patterns-handler.ts`, `consolidate-handler.ts`
    (PROTECTED — `remote = false` required)
  - `contradiction-handler.ts`, `calibration-handler.ts`

### 10.2 — `src/core/worker-pool.ts`

Thread-pool for parallel job execution.
`check-worker-pool-atomicity.sh` enforces: no double-write patterns.

---

## Phase 11 — Dream Cycle [§11]

### 11.1 — `src/core/cycle.ts`

The dream cycle orchestrator. Runs all phases in dependency order:
`sync → embed → extract → enrich → consolidate → synthesize → patterns
→ contradictions → calibration`

Each phase:
- Checks cost budget before starting
- Uses `op-checkpoint.ts` for crash-safe resumption
- Emits progress events (`src/core/progress.ts`)
- Writes audit records (`src/core/audit/audit-writer.ts`)

### 11.2 — `src/commands/autopilot.ts`

`gbrain autopilot --install --repo <path>`:
Writes cron jobs that invoke `gbrain cycle` on the configured schedule.
`--quiet-hours` config respected.

---

## Phase 12 — MCP Server [§9]

### 12.1 — `src/mcp/server.ts`

Sets `OperationContext.remote = true` for all operations (untrusted callers).
Registers all non-`localOnly` operations as MCP tools.
Stdio transport via `@modelcontextprotocol/sdk`.

### 12.2 — `src/mcp/http-transport.ts` [§9 Auth]

Express 5 HTTP server:
- `/mcp` — MCP JSON-RPC endpoint
- `/ingest` — webhook ingestion endpoint
- `/admin` — serves embedded admin SPA
- OAuth 2.1 + PKCE: `src/core/oauth-provider.ts`
- Scope enforcement before op dispatch
- Rate limiting: `src/mcp/rate-limit.ts`
- `requireAdmin` middleware for `/admin` and admin-scope ops

### 12.3 — `src/mcp/dispatch.ts`

Routes MCP tool calls to operation handlers. Enforces `scope` annotation.
Applies `sourceScopeOpts(ctx)` for source-isolated reads.

### 12.4 — `src/mcp/tool-defs.ts`

Generates MCP tool schemas from `src/core/operations.ts` operation definitions.
This is the single source for the MCP tool manifest.

---

## Phase 13 — CLI Commands [§9]

One file per command in `src/commands/`. Key commands to implement:

| Command | File | Description |
|---------|------|-------------|
| `init` | `init.ts` | Brain initialization wizard (PGLite / Postgres) |
| `doctor` | `doctor.ts` | Health check + remediation plan |
| `search` | `search.ts` | Hybrid search with `--explain` |
| `think` | `think.ts` | Synthesis query |
| `capture` | `capture.ts` | Quick page capture |
| `import` | `import.ts` | Bulk markdown import |
| `export` | `export.ts` | Bulk export |
| `sync` | `sync.ts` | Filesystem ↔ DB sync |
| `extract` | `extract.ts` | Fact/takes extraction |
| `embed` | `embed.ts` | Embedding backfill |
| `enrich` | `enrich.ts` | Entity enrichment |
| `serve` | `serve.ts` | stdio MCP server |
| `serve --http` | `serve-http.ts` | HTTP MCP + admin |
| `autopilot` | `autopilot.ts` | Cron install |
| `config` | `config.ts` | Config CRUD |
| `schema` | `schema.ts` | Schema pack management |
| `graph-query` | `graph-query.ts` | Multi-hop graph traversal |
| `upgrade` | `upgrade.ts` | Self-update + migrations |
| `apply-migrations` | `apply-migrations.ts` | Manual schema migrations |
| `recall` | `recall.ts` | Fact recall |
| `brainstorm` | `brainstorm.ts` | Ideation cycle |
| `dream` | `dream.ts` | Manual dream cycle trigger |
| `jobs` | `jobs.ts` | Job queue management |
| `whoknows` | `whoknows.ts` | Expert routing |
| `agent` | `agent.ts` | Sub-agent launch |
| `backfill` | `backfill.ts` | Data backfill ops |
| `pages` | `pages.ts` | Page listing |
| `frontmatter` | `frontmatter.ts` | Frontmatter management |
| `report` | `report.ts` | Brain reports |
| `eval` | `eval.ts` | Eval harness entrypoint |
| `models` | `models.ts` | List available AI models |
| `integrations` | `integrations.ts` | List integration recipes |

Also implement: `migrations/` sub-directory with versioned migration commands
(v0.11.0 → v0.32.2, matching `src/core/migrate.ts`).

---

## Phase 14 — Schema Pack System [§5]

### 14.1 — `src/core/schema-pack/`

- `primitives.ts` — `PackPrimitive` enum: `entity | media | temporal | annotation | concept`
- `index.ts` — `loadActivePack()`, `SchemaPackManifest` type
- `base/gbrain-base.yaml` — generated by `scripts/generate-gbrain-base.ts`
  from `ALL_PAGE_TYPES` + gbrain.yml config
- `base/gbrain-base-v2.yaml` — 15-type DRY/MECE taxonomy (default as of v0.41.22)
- `base/gbrain-recommended.yaml` — extends base with 13 additional directories

### 14.2 — `src/core/schema-embedded.ts`

Embeds pack YAML files as string literals for binary distribution
(same pattern as `src/admin-embedded.ts`).

### 14.3 — `scripts/generate-gbrain-base.ts`

Codegen: reads `ALL_PAGE_TYPES` + config and writes `gbrain-base.yaml`.

---

## Phase 15 — Audit & Observability [§17]

### 15.1 — `src/core/audit/audit-writer.ts`

Shared JSONL audit primitive:
`createAuditWriter({kind, recordSchema})` → `{log, readRecent}`.
- `computeIsoWeekFilename(kind, now?)` — ISO-week file rotation
- `resolveAuditDir()` — honors `GBRAIN_AUDIT_DIR` env var
- Best-effort writes (never throws)

All audit modules use this primitive:
`rerank-audit.ts`, `audit-slug-fallback.ts`, `shell-audit.ts`,
`supervisor-audit.ts`, `phantom-audit.ts`, `graph-signals-failures`.

### 15.2 — `src/core/progress.ts`

Progress event schema (JSONL to stdout):
```json
{"type": "progress", "phase": "embed", "done": 42, "total": 100, "pct": 42}
```
Enforced by `check-progress-to-stdout.sh`: progress never goes to stderr.

---

## Phase 16 — Admin SPA [§13]

### 16.1 — `admin/` React application

Scaffold with Vite + React 18:
```bash
cd admin && bun create vite . --template react-ts
```

Implement pages from §13:
- `Dashboard.tsx`, `Calibration.tsx`, `RequestLog.tsx`, `Logs.tsx`,
  `Auth.tsx`, `Health.tsx`

Apply design tokens from §13 design tokens table to `admin/src/index.css`.

`TrustedSVG.tsx` wrapper: renders server SVG via `dangerouslySetInnerHTML`
(safe: SVG is server-generated and XSS-escaped).

### 16.2 — `scripts/build-admin-embedded.ts`

Post-build step: reads every file in `admin/dist/`, emits `src/admin-embedded.ts`
with one `import x from './path' with { type: 'file' }` line per asset plus a
manifest map keyed by HTTP request path (`/admin/index.html`, `/admin/assets/*.js`).

`scripts/check-admin-embedded.sh` — CI guard: regenerates and `git diff --exit-code`.

---

## Phase 17 — Skills [§8]

### 17.1 — Skill file structure

Each skill lives in `skills/<skill-name>/SKILL.md`. Format:

```markdown
# <Skill Name>

## Trigger
<when the agent should use this skill>

## Protocol
<step-by-step instructions for the agent>

## Brain operations used
<list of gbrain CLI commands or MCP ops>

## Output format
<what the agent should return>
```

### 17.2 — Create all skills from §8

For each skill in §8, create `skills/<name>/SKILL.md` with:
- Trigger description matching `skills/RESOLVER.md` routing table
- Protocol steps using `gbrain` CLI commands or MCP ops
- Brain-first pattern: always search brain before external API calls
- Tool-agnostic markdown (works with Claude Code, Cursor, OpenClaw)

### 17.3 — `skills/RESOLVER.md`

The dispatcher. Structured as a table:

```markdown
| Trigger phrase | Skill |
|---------------|-------|
| ... | skills/<name>/SKILL.md |
```

Covers: always-on, brain ops, ingestion, query, operations, setup, identity/access.

### 17.4 — Shared convention files

```
skills/conventions/
  brain-routing.md     # two-axis (brain + source) routing decision table
  search-modes.md      # cost matrix for conservative/balanced/tokenmax
skills/_AGENT_README.md
skills/_brain-filing-rules.md
skills/_brain-filing-rules.json
skills/_output-rules.md
skills/_friction-protocol.md
skills/manifest.json   # skill metadata registry
```

---

## Phase 18 — Templates [§15]

Create four files in `templates/`:

```
templates/
  ACCESS_POLICY.md.template
  SOUL.md.template
  USER.md.template
  HEARTBEAT.md.template
```

Each is a markdown template with `{{VARIABLE}}` placeholders populated at `gbrain init`.
`SOUL.md.template` includes agent name, voice, personality, priorities.
`USER.md.template` includes user name, role, goals, relationships, communication style.

---

## Phase 19 — Recipes & Integrations [§16]

Create one markdown file per integration in `recipes/`:
- Filename: `<name>.md` (or `<name>/` directory for multi-file recipes)
- Format: "## What it does", "## Setup", "## Automation steps", "## Testing"
- Each discoverable via `gbrain integrations list` (reads `recipes/` directory)

Create `recipes/agent-voice/` for multi-file recipe with agent persona setup.

---

## Phase 20 — Eval Framework [§18]

### 20.1 — `src/eval/longmemeval/`

`harness.ts` — runs LongMemEval dataset against isolated per-question PGLite instances.
`adapter.ts` — maps dataset format to gbrain ingestion + query.
`intent.ts` — query intent classification.
`sanitize.ts` — scrub PII from eval output.
`extract.ts` — extract answers from synthesis output.

### 20.2 — `src/eval/retrieval-quality/`

`harness.ts` — NamedThingBench: title-substring, alias-synonym, generic-to-named,
multi-chunk-dilution families. Hard CI gate.

### 20.3 — `src/eval/code-retrieval/`

`harness.ts` + `strategies.ts` — code retrieval benchmark.

### 20.4 — `src/commands/eval.ts` and sub-commands [§18]

`gbrain eval <sub>`:
- `longmemeval` — LongMemEval benchmark
- `retrieval-quality` — NamedThingBench (CI gate)
- `cross-modal` — 3-model ensemble quality check
- `export` — NDJSON capture from `eval_candidates` table
- `replay` — compare captured queries against current brain
- `brainstorm` — brainstorm quality
- `gate` — regression gate (fails CI if P@5 drops > threshold)
- `compare` — compare two eval export files
- `run-all` — run all evals in sequence

---

## Phase 21 — Tests [§t]

### 21.1 — Test helpers

`test/helpers/`:
- `legacy-embedding-preload.ts` — pins gateway to OpenAI/1536 before any test
- `pglite-hermetic.ts` — `createHermeticEngine()` using basis-vector embeddings
- `withEnv.ts` — `withEnv(overrides, fn)` for env-scoped test isolation
- `basis-vectors.ts` — deterministic pseudo-random Float32Array for tests

### 21.2 — Unit test pattern

One `.test.ts` file per source module, co-located or in `test/`:
- Hermetic: no real API calls, no real DB connections
- Deterministic: basis vectors instead of real embeddings
- Isolated: `withEnv` for env overrides; each test gets its own PGLite instance
- Privacy: no real names in fixtures (enforced by `check-privacy.sh`)

### 21.3 — E2E tests (`test/e2e/`)

29 files requiring fresh Postgres (via `docker-compose.test.yml`):
- `engine-parity.test.ts` — verifies PGLite ≡ Postgres for every engine method
- `search-quality.test.ts` — P@5 / R@5 regression test
- `multi-source-bug-class.test.ts` — source isolation regression
- `graph-signals-engine.test.ts` — graph boost parity

### 21.4 — Regression tests (`test/regressions/`)

Named after the bug they pin, e.g. `v0_36_frontier_cap.test.ts`.

---

## Phase 22 — Scripts & CI Checks [§17]

### 22.1 — Build scripts

| Script | Purpose |
|--------|---------|
| `scripts/build-admin-embedded.ts` | Embed admin SPA into binary |
| `scripts/build-llms.ts` | Generate `llms.txt` + `llms-full.txt` |
| `scripts/build-pglite-snapshot.ts` | Pre-warm PGLite snapshot |
| `scripts/build-schema.sh` | Generate schema pack YAMLs |

### 22.2 — Test runner scripts

| Script | Purpose |
|--------|---------|
| `scripts/run-unit-parallel.sh` | Sharded parallel unit tests |
| `scripts/run-unit-shard.sh` | Single shard runner |
| `scripts/run-e2e.sh` | Sequential E2E tests (requires Postgres) |
| `scripts/run-slow-tests.sh` | Slow/heavy unit tests |
| `scripts/run-verify-parallel.sh` | Type-check + lint parallel |
| `scripts/ci-local.sh` | Full local CI gate |
| `scripts/select-e2e.ts` | Diff-aware E2E selector |

### 22.3 — Privacy & security check scripts [§17]

Create shell scripts enforcing each constraint in §17:

```
scripts/check-privacy.sh                   # no real names in test fixtures
scripts/check-test-real-names.sh           # broader name grep
scripts/check-no-pii-in-agent-voice.sh     # no PII in voice output
scripts/check-source-config-leak.sh        # no source config in output
scripts/check-operations-filter-bypass.sh  # ops filter always applied
scripts/check-gateway-routed-no-direct-anthropic.sh  # all AI via gateway
scripts/check-worker-pool-atomicity.sh     # no double-write
scripts/check-no-double-retry.sh           # no nested retry
scripts/check-admin-scope-drift.sh         # admin ops stay admin-scoped
scripts/check-jsonb-pattern.sh             # safe JSONB query patterns
scripts/check-source-id-projection.sh      # source_id always projected
scripts/check-progress-to-stdout.sh        # progress events to stdout
scripts/check-test-isolation.sh            # hermetic test isolation
scripts/check-trailing-newline.sh          # files end with newline
scripts/check-wasm-embedded.sh             # WASM grammars embedded
scripts/check-exports-count.sh             # exports count stable
scripts/check-admin-build.sh               # admin SPA builds cleanly
scripts/check-admin-embedded.sh            # embedded module up to date
scripts/check-cli-executable.sh            # CLI binary is executable
scripts/check-skill-brain-first.sh         # skills use brain before external API
scripts/check-batch-audit-site.sh          # batch audit call sites
scripts/check-worker-lock-renewal-shape.sh # lock renewal pattern
scripts/check-system-of-record.sh          # system-of-record annotations
scripts/check-proposal-pii.sh              # no PII in proposals
scripts/check-synthetic-corpus-privacy.sh  # synthetic corpus is clean
scripts/check-fuzz-purity.sh               # fuzz tests are pure
scripts/check-source-scope-onboard.sh      # source scope on onboard
```

---

## Phase 23 — Documentation

### 23.1 — Root documentation files

| File | Content |
|------|---------|
| `README.md` | Installation (agent-driven, CLI, MCP), query examples, capabilities, integrations, tutorials |
| `AGENTS.md` | Agent install protocol (non-Claude-Code); read order; trust boundary; common tasks |
| `CLAUDE.md` | Claude Code operating protocol; architecture reference; key files; test layout |
| `INSTALL_FOR_AGENTS.md` | 9-step agent install guide (keys, init, search mode, cron, verify) |
| `DESIGN.md` | Design system: voice, color tokens, typography, spacing, layout, charts |
| `CONTRIBUTING.md` | Contribution guide: branch naming, PR process, CI gates |
| `SECURITY.md` | Security policy and responsible disclosure |
| `CHANGELOG.md` | Versioned change log |
| `TODOS.md` | Known technical debt + future work items |
| `llms.txt` | Documentation map for LLM consumption |
| `llms-full.txt` | Same map with core docs inlined |

### 23.2 — `docs/` directory structure

```
docs/
  INSTALL.md                      # Multi-engine setup
  ENGINES.md                      # Engine config and tuning
  GBRAIN_VERIFY.md                # Verification checklist
  GBRAIN_RECOMMENDED_SCHEMA.md    # Entity schema guide
  GBRAIN_SKILLPACK.md             # Skillpack documentation
  UPGRADING_DOWNSTREAM_AGENTS.md  # Upgrade patches
  architecture/
    brains-and-sources.md         # Two-axis mental model
    topologies.md                 # Brain shape topologies
    schema-packs.md               # Schema pack authoring
    pack-upgrade-mechanism.md
  eval/
    SEARCH_MODE_METHODOLOGY.md    # Eval methodology
  guides/
    live-sync.md
    quiet-hours.md
    skill-development.md
    minions-fix.md
    skillopt.md
    sub-agent-routing.md
  mcp/
    CLAUDE_CODE.md
    CLAUDE_DESKTOP.md
    CLAUDE_COWORK.md
    PERPLEXITY.md
    CHATGPT.md
    DEPLOY.md
  tutorials/
    personal-brain.md             # Personal brain end-to-end
    company-brain.md              # Team/company brain
    improving-skills-with-skillopt.md
  ethos/
    THIN_HARNESS_FAT_SKILLS.md
    MARKDOWN_SKILLS_AS_RECIPES.md
  what-schemas-unlock.md
  schema-author-tutorial.md
  skillpack-anatomy.md
  takes-vs-facts.md
  contradictions.md
  guardrails.md
```

---

## Phase 24 — OpenClaw / Plugin Integration

### 24.1 — `src/openclaw-context-engine.ts`

Exports `gbrainContextEngine`: implements OpenClaw's `ContextEngine` contract.
Returns brain search results as agent context for the OpenClaw platform.

### 24.2 — `src/admin-embedded.ts` (generated)

Do NOT write by hand. Generated by `scripts/build-admin-embedded.ts` after
`bun run build:admin`. CI guard prevents divergence.

---

## Phase 25 — Final Wiring & Verification

### 25.1 — Run CI checks

```bash
bun run verify          # typecheck + lint
bun run check:all       # all privacy + quality checks
bun run test            # unit tests (parallel)
bun run test:e2e        # E2E (requires docker-compose.test.yml)
```

All checks must pass. Fix any failures before shipping.

### 25.2 — Doctor check

```bash
gbrain init --pglite
gbrain doctor
```

All doctor checks should be ✓ or have expected warnings for unconfigured API keys.

### 25.3 — Smoke test

```bash
gbrain capture "test thought"
gbrain search "test"
gbrain think "what do I know?"
```

### 25.4 — Verify MCP

```bash
gbrain serve &
# In Claude Code: claude mcp add gbrain -- gbrain serve
```

---

## Checklist: Domain Swap Procedure

To regenerate this repo for a **different domain**, follow this exact procedure:

1. **Edit `CONTEXT_REVERSE.md` only:**
   - §1: Change project name, binary name, GitHub slug
   - §2: Rewrite the domain + target user profile
   - §5: Replace entity types and path prefixes with domain-specific ones
   - §8: Add/remove/rename skills for the new domain
   - §12: Adjust calibration if bias learning is relevant
   - §15: Rewrite `SOUL.md.template` and `USER.md.template` for new persona
   - §16: Replace recipes with domain-relevant integrations
   - §18: Adjust eval benchmarks for new domain

2. **Do NOT edit `PLAN_REVERSE.md`** — it is domain-agnostic.

3. **Re-execute `PLAN_REVERSE.md`** against the updated `CONTEXT_REVERSE.md`.

4. The result is a fully functional knowledge brain for the new domain.

---

## Invariants (Never Change Across Domains)

These architectural decisions are load-bearing and must be preserved in any regeneration:

- **Contract-first operations** (`src/core/operations.ts` as single source of truth)
- **Pluggable engines** (PGLite + Postgres both implement `BrainEngine`)
- **Trust boundary** (`OperationContext.remote` is REQUIRED, fail-closed)
- **All AI via gateway** (`src/core/ai/gateway.ts`, no direct provider imports)
- **Hermetic tests** (no real API calls, basis-vector embeddings)
- **Progress to stdout** (never stderr)
- **Audit via JSONL** (ISO-week rotation, best-effort writes)
- **Source isolation** (`sourceScopeOpts(ctx)` applied on every read-side op)
- **Admin SPA embedded** (generated by `build-admin-embedded.ts`, CI-guarded)
- **Per-page max-pool** (vector search surfaces page on its strongest chunk)
- **Floor-ratio gate** (graph boosts cannot elevate weak pages past strong ones)
