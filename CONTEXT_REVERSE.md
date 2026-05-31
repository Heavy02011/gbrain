# CONTEXT_REVERSE.md

> **What this file is:** All domain-specific, user-specific, and deployment-specific
> decisions that shape a GBrain-class knowledge brain. Every item here is a variable.
> Change any section and `PLAN_REVERSE.md` will produce a different brain for a
> different domain, user, or stack. Nothing in `PLAN_REVERSE.md` needs to change;
> only this file changes.

---

## 1. Project Identity

| Key | Value |
|-----|-------|
| **Project name** | `gbrain` |
| **npm/bun package name** | `gbrain` |
| **Binary name** | `gbrain` |
| **Version** | `0.42.1.0` |
| **License** | MIT |
| **Description** | Postgres-native personal knowledge brain with hybrid RAG search |
| **Author / origin** | Garry Tan, YC |
| **GitHub slug** | `garrytan/gbrain` |

---

## 2. Domain & Target User

**Domain:** Personal + organizational knowledge management for a high-volume operator
(CEO / VC / researcher) who accumulates 100K+ pages across meetings, people, companies,
ideas, tweets, emails, and original writing.

**Primary user profile:**
- Ingests: meetings, voice calls, emails, tweets, articles, books, PDFs, research
- Queries: "Who works at X?", "What did I promise Y?", "Who knows Alice?", "What's open?"
- Need: synthesized answers with citations AND gap analysis (not just chunk ranking)
- Scale: 146K pages, 24K people, 5K companies, 66 cron jobs

**To adapt for another domain:** replace the user profile, entity types, and skills
list (Sections 5, 6, 8) with the new domain's equivalents:
- Legal brain → `case`, `motion`, `deponent`, `exhibit`, `counsel` types
- Research brain → `paper`, `experiment`, `hypothesis`, `dataset`, `lab-result` types
- Medical brain → `patient`, `encounter`, `prescription`, `lab-result` types
- Sales brain → `lead`, `opportunity`, `account`, `call-summary`, `deal` types

---

## 3. Runtime & Build Stack

| Component | Choice | Notes |
|-----------|--------|-------|
| **Runtime** | Bun ≥ 1.3.10 | Required; no Node fallback |
| **Language** | TypeScript (strict) | `tsconfig.json` targets ESNext |
| **Module system** | ESM (`"type": "module"`) | Bun-native |
| **Build tool** | `bun build --compile` | Produces standalone binary |
| **Admin frontend** | Vite + React 18 | Embedded into binary |
| **Test framework** | `bun test` | Hermetic via PGLite + basis vectors |
| **Timeout** | 60 000 ms | PGLite cold-start is slow |

---

## 4. Database Engines

Two interchangeable engines, selected at init time:

| Engine | When | Notes |
|--------|------|-------|
| **PGLite** (`@electric-sql/pglite@0.4.3`) | Default, < 1000 files, zero-config | WASM Postgres, no server |
| **Postgres + pgvector** | ≥ 1000 files, multi-machine | Supabase recommended |

Both engines implement the same `BrainEngine` interface (`src/core/engine.ts`).
Engine is selected at runtime by `engine-factory.ts` via `config.engine` key.

**PGLite snapshot path:** `~/.gbrain/pglite/` (default).
**Postgres URL:** env `DATABASE_URL` or `~/.gbrain/config.json` key `database_url`.

---

## 5. Entity Schema (Schema Pack: `gbrain-base-v2`)

The active schema pack is **`gbrain-base-v2`** (default as of v0.41.22).

### Canonical page types

| Type | Path prefix | Extractable | Expert routing |
|------|-------------|-------------|---------------|
| `person` | `people/` | ✓ | ✓ |
| `company` | `companies/` | ✓ | ✓ |
| `deal` | `deals/` | ✓ | ✓ |
| `meeting` | `meetings/` | ✓ | – |
| `note` | `notes/`, `inbox/` | – | – |
| `concept` | `concepts/` | ✓ | – |
| `analysis` | `analysis/` | ✓ | – |
| `atom` | `atoms/` | – | – |
| `media` | `media/` | – | – |
| `tweet` | `media/x/` | – | – |
| `social-digest` | `media/digests/` | – | – |
| `source` | `sources/` | – | – |
| `email` | `inbox/email/` | – | – |
| `slack` | `inbox/slack/` | – | – |
| `writing` | `writing/` | ✓ | – |
| `project` | `projects/` | ✓ | ✓ |
| `code` | `code/` | – | – |
| `image` | `images/` | – | – |
| `synthesis` | `synthesis/` | – | – |
| `conversation` | `conversations/` | – | – |
| `extract_receipt` | `extracts/` | – | – |

### Typed graph edges

`attended` · `works_at` · `invested_in` · `founded` · `advises` · `mentions`
· `authored` · `part_of` · `linked_from` · `cites`

### gbrain.yml (storage tiers)

```yaml
storage:
  db_tracked:        # git-versioned, human-curated
    - people/
    - companies/
    - deals/
    - concepts/
    - yc/
    - ideas/
    - projects/
  db_only:           # db-only, machine-generated, .gitignored
    - media/x/
    - media/articles/
    - meetings/transcripts/
```

**To adapt:** add, rename, or remove types and prefixes to match the target domain.
The `generate-gbrain-base.ts` script regenerates the pack YAML from this seed list.

---

## 6. AI Providers & Model Routing

### Embedding stack (default as of v0.36.2.0)

| Role | Provider / Model | Dims | Key env var |
|------|-----------------|------|------------|
| **Embedding** (default) | ZeroEntropy | 1280 | `ZEROENTROPY_API_KEY` |
| **Reranker** (default) | ZeroEntropy | – | `ZEROENTROPY_API_KEY` |
| **Embedding** (fallback) | OpenAI `text-embedding-3-small` | 1536 | `OPENAI_API_KEY` |
| **Embedding** (alt) | Voyage | 1024 | `VOYAGE_API_KEY` |

### Chat / synthesis models

| Role | Default model | Fallback | Key env var |
|------|--------------|---------|------------|
| **Query expansion** | Anthropic Haiku | skip if absent | `ANTHROPIC_API_KEY` |
| **Synthesis / think** | Anthropic Sonnet | – | `ANTHROPIC_API_KEY` |
| **Enrichment / extract** | Anthropic Sonnet | OpenAI GPT-4o | `ANTHROPIC_API_KEY` |
| **Voice judge** | Anthropic Haiku | – | – |
| **Cross-modal judge** | 3-model ensemble | Anthropic + Google + OpenAI | all three keys |
| **Image captioning** | Anthropic vision | – | – |

### Supported provider recipes (in `src/core/ai/recipes/`)

Anthropic · OpenAI · Azure OpenAI · Google Gemini · Groq · Ollama · LiteLLM proxy
· llama-server · ZeroEntropy · OpenRouter · Together · DeepSeek · Minimax · Zhipu
· DashScope · Voyage

**All provider calls route through `src/core/ai/gateway.ts`** — no direct provider
imports in feature code. This is enforced by `check-gateway-routed-no-direct-anthropic.sh`.

---

## 7. Search Configuration

### Hybrid search knobs

| Mode | Vector | Keyword | Graph signals | Reranker | Notes |
|------|--------|---------|--------------|---------|-------|
| `conservative` | ✓ | ✓ | off | off | lowest cost |
| `balanced` | ✓ | ✓ | ✓ | ZeroEntropy | **default** |
| `tokenmax` | ✓ | ✓ | ✓ | ZeroEntropy | highest quality, most tokens |

### Retrieval pipeline stages

1. Vector search (HNSW on pgvector, per-page max-pool)
2. BM25 keyword search
3. Reciprocal-rank fusion (RRF)
4. Source-tier boost (per source type weight)
5. Backlink boost + salience boost + recency boost + exact-match boost
6. Graph adjacency signals (adjacency 1.05×, cross-source 1.10×, session demote 0.95×)
7. ZeroEntropy reranker (balanced + tokenmax only)

### Search cache

`query_cache` table, TTL 3600s, cache-key includes mode + knobs hash.

---

## 8. Skills (Agent Capability Set)

Skills are Markdown files, tool-agnostic, installed into the agent workspace.
This is the full set bundled with the default install:

### Core (always-on)
- `signal-detector` — fires on every inbound message
- `brain-ops` — any brain read/write/lookup

### Ingestion
- `capture`, `idea-ingest`, `ingest`, `media-ingest`, `meeting-ingestion`
- `voice-note-ingest`, `book-mirror`, `brain-pdf`, `archive-crawler`
- `webhook-transforms`

### Enrichment & graph
- `enrich`, `data-research`, `citation-fixer`, `concept-synthesis`
- `strategic-reading`, `academic-verify`, `article-enrichment`

### Query & synthesis
- `query`, `briefing`, `daily-task-prep`, `reports`
- `cross-modal-review`, `perplexity-research`

### Operations
- `maintain` (dream cycle, health, consolidation)
- `daily-task-manager`, `cron-scheduler`
- `minion-orchestrator`, `soul-audit`
- `skillpack-check`, `smoke-test`
- `publish`, `brain-taxonomist`, `repo-architecture`

### Dev / meta
- `skill-creator`, `skillify`, `skillpack-harvest`, `skill-optimizer`
- `functional-area-resolver`, `ask-user`
- `schema-author`, `schema-unify`, `testing`
- `setup`, `cold-start`, `migrate`, `install`
- `eiirp` (Everything In Its Right Place)
- `frontmatter-guard`

### Conventions (shared deps, not standalone)
- `conventions/brain-routing.md`
- `conventions/search-modes.md`
- `_AGENT_README.md`, `_brain-filing-rules.md`, `_output-rules.md`

**To adapt for a new domain:** swap or remove skills. Every skill is a single
`SKILL.md` file in its own directory. The resolver (`skills/RESOLVER.md`) maps
trigger phrases to skill files.

---

## 9. MCP Server

### Transports
- **stdio** (`gbrain serve`) — for Claude Code, Cursor, Windsurf
- **HTTP** (`gbrain serve --http`) — for Claude Desktop, Cowork, Perplexity, ChatGPT

### Auth (HTTP only)
- OAuth 2.1 + PKCE
- DCR-style client registration
- Scopes: `read` · `write` · `admin`
- Confidential client SHA-256 hashing
- Per-client rate limiting

### Operations exposed (47+)

`put_page` · `get_page` · `list_pages` · `search` · `query` · `think`
· `find_experts` · `find_trajectory` · `find_contradictions` · `find_anomalies`
· `graph_traversal` · `submit_job` · `get_job` · `list_jobs` · `whoami`
· `get_recent_salience` · `get_recent_transcripts` · `schema_apply_mutations`
· `file_upload` · `file_list` · `file_url` · `sync_brain` (admin+localOnly)
… and 20+ more. Full list in `src/core/operations.ts`.

---

## 10. Job Queue (Minions)

BullMQ-shaped, Postgres-native. Key characteristics:
- Two-phase persistence: `pending → done` (crash-safe)
- Child jobs with cascading timeouts
- Rate leases for outbound API providers
- S3/Supabase attachments
- Protected job names (only trusted callers can submit): `synthesize`, `patterns`, `consolidate`
- Shell jobs: sandboxed to trusted local callers only

Queue table: `minion_jobs`.

---

## 11. Dream Cycle (Autonomous Maintenance)

Runs as background cron jobs. Key phases:

| Phase | What it does | Cost tier |
|-------|-------------|-----------|
| `sync` | Watch filesystem changes, sync to DB | free |
| `embed` | Generate embeddings for new/stale chunks | per-call |
| `extract` | Extract facts, takes, timeline entries | LLM |
| `enrich` | Enrich people/company pages via web research | LLM + web |
| `consolidate` | Dedup, merge, fix citations | LLM |
| `synthesize` | Compose synthesis pages | LLM (PROTECTED) |
| `patterns` | Find recurring themes | LLM (PROTECTED) |
| `contradictions` | Surface conflicting facts | LLM |
| `calibration` | Update user bias model from graded takes | LLM |

**Install cron:** `gbrain autopilot --install --repo ~/brain`

---

## 12. Calibration System

Learns the user's epistemic biases through graded "takes" (short predictions/opinions).

- Users grade takes after outcomes resolve
- System fits a Brier-score calibration model per domain
- Applies anti-bias rewrites in synthesis output
- Visualized in admin SPA (Calibration tab: domain bars, Brier trend)
- 6 DB migrations (v67–v72) for the calibration data model

---

## 13. Admin SPA

React 18 + Vite, dark-theme only, embedded into the CLI binary.
Served at `http://localhost:8000/admin` when `gbrain serve --http`.

### Pages
- Dashboard (brain identity, page count, health score)
- Calibration (bias profiles, Brier chart, domain aggregates)
- Request Log (API audit, per-user spend)
- Logs (system log viewer)
- Auth (OAuth client registration, scope management)
- Health (doctor output, remediation plan)

### Design tokens (from `DESIGN.md`)

| Token | Value | Use |
|-------|-------|-----|
| `--bg-primary` | `#0a0a0f` | Page background |
| `--bg-secondary` | `#14141f` | Sidebar, cards |
| `--bg-tertiary` | `#1e1e2e` | Subtle surfaces |
| `--text-primary` | `#e0e0e0` | Body text |
| `--text-secondary` | `#888` | Headings, labels |
| `--text-muted` | `#777` | Tertiary text |
| `--accent` | `#3b82f6` | Active states, links |
| `--success` | `#22c55e` | Healthy status |
| `--warning` | `#f59e0b` | Warnings |
| `--error` | `#ef4444` | Failures |

Font: `Inter` (UI) · `JetBrains Mono` (numbers, slugs).
Spacing scale: 4 / 8 / 16 / 24 / 32 px. Sidebar: 200 px.

**Charts:** server-rendered SVG (pure functions, no DOM), XSS-safe via `escapeXml()`.

---

## 14. Deployment Targets

| Target | Command | Notes |
|--------|---------|-------|
| Local PGLite | `gbrain init --pglite` | Default, zero-config |
| Local Postgres | `gbrain init --postgres` | pgvector required |
| Supabase | `gbrain init --supabase` | Recommended for scale |
| OpenClaw (Render) | one-click deploy | 8 GB RAM required |
| Hermes (Railway) | one-click deploy | |
| ngrok tunnel | `gbrain serve --http` + ngrok | For remote MCP clients |
| Docker (CI) | `docker-compose.ci.yml` | E2E tests only |

---

## 15. Templates (Agent Identity Files)

Four markdown templates instantiated at `gbrain init` in the brain root:

| File | Purpose |
|------|---------|
| `ACCESS_POLICY.md` | Who can message the agent; what they can ask |
| `SOUL.md` | Agent's name, voice, personality, priorities |
| `USER.md` | User context: role, relationships, goals |
| `HEARTBEAT.md` | Operational cadence: what to check and when |

These are the agent's "identity layer." Changing `SOUL.md` changes the agent's persona.

---

## 16. Integrations / Recipes

Shipped in `recipes/` as markdown + setup hints, discoverable via `gbrain integrations list`.

| Recipe | What it does |
|--------|-------------|
| `x-to-brain.md` | Mirror Twitter/X posts |
| `email-to-brain.md` | Ingest email via Zapier / IFTTT |
| `calendar-to-brain.md` | Ingest calendar events |
| `meeting-sync.md` | Auto-ingest meeting transcripts (Granola, Otter) |
| `twilio-voice-brain.md` | Voice calls → transcripts → brain |
| `ngrok-tunnel.md` | Expose HTTP MCP to remote clients |
| `credential-gateway.md` | Secure API key management |
| `restart-sweep.md` | Post-crash health sweep |
| `agent-voice.md` | Agent voice/persona setup |

---

## 17. Privacy & Security Constraints

These are enforced by CI check scripts and must be preserved in any regeneration:

1. **No real names in test fixtures** — `check-privacy.sh`, `check-test-real-names.sh`
2. **No PII in agent voice output** — `check-no-pii-in-agent-voice.sh`
3. **No source config leaks** — `check-source-config-leak.sh`
4. **Operations filter never bypassed** — `check-operations-filter-bypass.sh`
5. **Gateway-routed only** — `check-gateway-routed-no-direct-anthropic.sh`
6. **Worker pool atomicity** — `check-worker-pool-atomicity.sh`
7. **No double retry** — `check-no-double-retry.sh`
8. **Admin scope drift** — `check-admin-scope-drift.sh`
9. **SSRF defense** — `ssrf-validate.ts` validates all URLs before fetch

**Trust boundary** (non-negotiable):
- `OperationContext.remote = false` → trusted local CLI caller
- `OperationContext.remote = true` → untrusted MCP/HTTP caller
- Default is fail-closed: anything that is not strictly `false` is treated as remote

---

## 18. Eval Framework

| Benchmark | Command | What it measures |
|-----------|---------|-----------------|
| LongMemEval | `gbrain eval longmemeval` | Long-memory Q&A on public dataset |
| NamedThingBench | `gbrain eval retrieval-quality` | Named-entity retrieval families |
| BrainBench | via `gbrain-evals` repo | 240-page Opus corpus; +31.4 P@5 |
| Replay bench | `gbrain eval export` + `gbrain eval replay` | Real captured queries vs. code changes |
| Cross-modal | `gbrain eval cross-modal` | 3-model ensemble quality gate |
| Brainstorm | `gbrain eval brainstorm` | Brainstorm output quality |
| Contradiction | `gbrain eval suspected-contradictions` | Conflict surface accuracy |

All eval harnesses use isolated hermetic PGLite instances — `~/.gbrain` is never opened.

---

## 19. Key Metrics (Baseline — gbrain-base v2 + ZeroEntropy reranker)

| Metric | Value | Comparison |
|--------|-------|-----------|
| P@5 | 49.1% | +31.4 pp over vector-only RAG |
| R@5 | 97.9% | — |
| Brier score improvement | measured per-user | calibration tracks over time |

---

## 20. Agent Voice & Tone

From `DESIGN.md`:
- Second person, contractions allowed
- Grounded in concrete, verifiable data
- Never preachy; never "we recommend"; never "according to your data"
- Under 25 words for narrative; under one line for status
- Numbers grounded in real outcomes, never abstract metrics without translation

Five surfaces: `pattern_statement`, `nudge`, `forecast_blurb`, `dashboard_caption`, `morning_pulse`.
All gated by `gateVoice()` in `src/core/calibration/voice-gate.ts`.
Fallback to hand-written templates in `src/core/calibration/templates.ts`.
