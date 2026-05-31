# GBrain prompts and skills cheatsheet

This cheatsheet summarizes the agent-facing skill system and the TypeScript prompt surfaces used by GBrain. It is intended as a quick routing reference for contributors who need to decide which skill, convention, command, or prompt entry point applies to a task.

## 1. Core operating model

### Dispatcher rule

- `skills/RESOLVER.md` is the top-level skill dispatcher. Start there when a task might involve a skill.
- Read the matching `SKILL.md` before acting.
- If two skills could match, read both. Skills are intentionally designed to chain.

### Always-on skills

- `skills/signal-detector/SKILL.md` applies to every inbound message. Use it for ambient detection of original thinking and entity mentions.
- `skills/brain-ops/SKILL.md` applies to brain read, write, lookup, and citation work. Use it for the brain-first lookup and read-enrich-write loop.

### Cross-cutting conventions

All brain-writing skills should respect these shared conventions:

- `skills/conventions/quality.md` — citations, backlinks, and notability gate.
- `skills/conventions/brain-first.md` — check the brain before external APIs.
- `skills/conventions/brain-routing.md` — route on both brain and source.
- `skills/conventions/schema-evolution.md` — decide when to add a type, alias, or prefix.
- `skills/conventions/subagent-routing.md` — decide when to use Minions versus inline work.
- `skills/ask-user/SKILL.md` — use explicit choice gates at decision points.
- `skills/_brain-filing-rules.md` — file brain pages in the right location.
- `skills/_output-rules.md` — meet output quality expectations.

### Routing mental model

- **Brain** means which database.
- **Source** means which repo inside that database.
- Every query routes on both axes.
- Default behavior is to start from the environment-resolved brain and source, trust mounts and dotfiles, and make routing visible by passing the resolved brain explicitly.
- Cross-brain federation is agent-mediated rather than a deterministic SQL fanout.

## 2. Skill dispatcher cheatsheet

### Always-on and brain operations

| User intent or trigger | Skill or action | What it does |
| --- | --- | --- |
| Every inbound message | `skills/signal-detector/SKILL.md` | Detects original thinking, entity mentions, and other ambient signals. |
| Any brain read, write, lookup, or citation | `skills/brain-ops/SKILL.md` | Performs brain-first lookup and read-enrich-write workflow. |
| "What do we know about...", "tell me about...", "search for...", "who is..." | `skills/query/SKILL.md` | Answers with layered search, synthesis, and citation propagation. |
| "Who knows who", "relationship between", "connections", graph query | `skills/query/SKILL.md` graph-query path | Looks up relationship and graph context. |
| Create or enrich person or company page | `skills/enrich/SKILL.md` | Uses tiered enrichment templates and validation rules. |
| "Where does this go?", filing rules | `skills/repo-architecture/SKILL.md` | Applies directory conventions for brain files. |
| "Where does this brain page go?", taxonomy check, refile | `skills/brain-taxonomist/SKILL.md` | Reads the active schema pack and recommends a filing path. |
| "EIIRP", "put this in the brain", "organize all this work" | `skills/eiirp/SKILL.md` | Runs the post-work organizer: inventory, taxonomy, schema, filing, skill graph, verify, report. |
| Citation audit or citation repair | `skills/citation-fixer/SKILL.md` | Audits and repairs brain-page citation formatting. |
| Data research, investor updates, donations, metrics | `skills/data-research/SKILL.md` | Runs structured search, extraction, archiving, dedupe, and tracking with YAML recipes. |
| Share brain page as link | `skills/publish/SKILL.md` | Publishes password-protected HTML with zero LLM calls. |
| Validate or fix frontmatter | `skills/frontmatter-guard/SKILL.md` | Validates and repairs YAML frontmatter. |
| Search mode, cache heat, retrieval tuning | `gbrain search modes`, `gbrain search stats`, `gbrain search tune` | Runs retrieval and search-mode operations directly. |
| Search benchmarks or regression checks | `gbrain eval run-all`, `gbrain eval compare` | Runs retrieval evaluation directly. |

## 3. Ingestion skills

| User intent or trigger | Skill | What it does |
| --- | --- | --- |
| "Capture this", "remember this", "save to brain" | `skills/capture/SKILL.md` | Human-facing ingestion entry point; routes through local `put_page` or MCP depending on context. |
| Link, article, tweet, idea | `skills/idea-ingest/SKILL.md` | Ingests links, articles, tweets, and ideas with analysis and entity cross-linking. |
| Video, YouTube, PDF, podcast, book, screenshot, repo | `skills/media-ingest/SKILL.md` | Ingests media, documents, and repos with entity extraction. |
| Meeting transcript | `skills/meeting-ingestion/SKILL.md` | Ingests transcripts with attendee enrichment, entity propagation, and timeline merge. |
| Generic "ingest this" | `skills/ingest/SKILL.md` | Detects input type and delegates to specialized ingestion skills. |
| Voice memo or voice note | `skills/voice-note-ingest/SKILL.md` | Preserves exact phrasing and routes to originals, concepts, people, companies, ideas, personal, or voice-notes. |
| Raw article dumps need cleanup | `skills/article-enrichment/SKILL.md` | Converts raw article text into structured pages with summary, quotes, insights, why-it-matters, and cross-references. |
| Personal archives, Dropbox, B2, email exports | `skills/archive-crawler/SKILL.md` | Finds high-value content in explicitly allow-listed personal archives. |

## 4. Reading, synthesis, and research skills

| User intent or trigger | Skill | What it does |
| --- | --- | --- |
| Personalized version of a book, "mirror this book" | `skills/book-mirror/SKILL.md` | Produces chapter-by-chapter two-column analysis: source content versus personalized application. |
| Strategic reading, "read this through lens of X" | `skills/strategic-reading/SKILL.md` | Produces an applied playbook for a specific strategic problem. |
| Concept synthesis or intellectual map | `skills/concept-synthesis/SKILL.md` | Dedupes concept stubs into a tiered intellectual map and traces idea evolution. |
| Current-state web research, "what changed?" | `skills/perplexity-research/SKILL.md` | Produces brain-augmented current-state research that separates new information from already-known context. |
| Academic claim or citation verification | `skills/academic-verify/SKILL.md` | Verifies academic citations and current literature, then writes a citation-checked brain page. |
| Brain page to PDF | `skills/brain-pdf/SKILL.md` | Renders a publication-quality PDF via `gstack make-pdf`, strips frontmatter, and sanitizes emoji. |

## 5. Operations and automation skills

| User intent or trigger | Skill or command | What it does |
| --- | --- | --- |
| Task add, remove, complete, defer, review | `skills/daily-task-manager/SKILL.md` | Manages task lifecycle. |
| Morning prep or day planning | `skills/daily-task-prep/SKILL.md` | Reviews calendar, open threads, and tasks. |
| Daily briefing | `skills/briefing/SKILL.md` | Produces meeting context, active deals, and citation-tracked daily briefing. |
| Cron scheduling | `skills/cron-scheduler/SKILL.md` | Manages schedules with staggering, quiet hours, and wake-up override. |
| Save or load reports | `skills/reports/SKILL.md` | Creates timestamped reports with keyword routing. |
| Create or improve skill | `skills/skill-creator/SKILL.md` | Creates conformant skills with MECE validation. |
| "Skillify this" | `skills/skillify/SKILL.md` | Turns raw feature work into a skilled, tested, resolvable, evaled unit. |
| Compress resolver or shrink AGENTS.md | `skills/functional-area-resolver/SKILL.md` | Replaces skill-per-row routing with functional-area dispatch. |
| Health check or morning skillpack check | `skills/skillpack-check/SKILL.md` | Produces an agent-readable health report wrapping doctor and migration checks. |
| Harvest or lift skill upstream | `skills/skillpack-harvest/SKILL.md` | Genericizes and scrubs host-specific skills before upstreaming. |
| Post-restart smoke test | `skills/smoke-test/SKILL.md` | Runs post-restart checks and auto-fix for GBrain/OpenClaw environments. |
| Second opinion or cross-modal review | `skills/cross-modal-review/SKILL.md` | Runs a quality gate via a second model and refusal routing chain. |
| Validate skills | `skills/testing/SKILL.md` | Validates frontmatter, required sections, manifest entries, and MECE routing. |
| Webhook setup or external events | `skills/webhook-transforms/SKILL.md` | Converts external events into brain-ingestible signals. |
| Spawn, background, parallel, or shell job | `skills/minion-orchestrator/SKILL.md` | Orchestrates durable observable jobs and LLM subagents. |
| Present options or ask before proceeding | `skills/ask-user/SKILL.md` | Uses a reusable two-to-four-option choice gate that stops until the user responds. |

## 6. Setup, migration, schema, and maintenance

| User intent or trigger | Skill or command | What it does |
| --- | --- | --- |
| Set up GBrain or first boot | `skills/setup/SKILL.md` | Auto-provisions Supabase/PGLite, injects AGENTS.md, and runs first import. |
| "Now what?", cold start, import my data | `skills/cold-start/SKILL.md` | Runs day-one bootstrapping for contacts, calendar, email, conversations, social, and archives. |
| Migrate from Obsidian, Notion, Logseq, etc. | `skills/migrate/SKILL.md` | Migrates from Obsidian, Notion, Logseq, markdown, CSV, JSON, and Roam. |
| Brain health or maintenance run | `skills/maintain/SKILL.md` | Checks backlinks, citations, filing, stale info, orphans, and benchmarks. |
| Feature score or brain score | `gbrain features --json` | Reports feature and health surfaces directly. |
| Autopilot or ongoing maintenance | `gbrain autopilot --install --repo ~/brain` | Installs recurring brain maintenance. |
| Agent identity or "who am I?" | `skills/soul-audit/SKILL.md` | Runs an interactive interview that generates identity and policy files. |
| Add or evolve page types/schema | `skills/schema-author/SKILL.md` | Evolves schema packs and wraps schema CLI/MCP operations. |
| Unify or collapse noisy types | `skills/schema-unify/SKILL.md` | Migrates to canonical taxonomy via a seven-phase workflow. |
| Optimize skills | `skills/skill-optimizer/SKILL.md` | Runs the SkillOpt loop with validation gating, safety, locks, and atomic writes. |

## 7. Always-on identity and access files

| Trigger | File to check | Why |
| --- | --- | --- |
| Non-owner sends a message | `ACCESS_POLICY.md` | Enforces access rules before responding. |
| Agent needs its identity or vibe | `SOUL.md` | Provides agent identity. |
| Agent needs user context | `USER.md` | Provides user context. |
| Agent needs operating cadence | `HEARTBEAT.md` | Provides what to check and when. |

## 8. Disambiguation rules

When multiple skills match:

1. Prefer the most specific skill.
2. Route URLs by content type.
3. If the user mentions a person or company, decide whether enrichment or query is the better fit.
4. Follow each skill's explicit chaining phases.
5. Ask the user when the route is still ambiguous.

## 9. TypeScript prompt inventory

The tables below summarize prompt surfaces in `src/` and `src/commands/`. These are separate from the markdown skills.

### Core answer and search prompts

| Prompt | File | Purpose |
| --- | --- | --- |
| Think synthesis system prompt | `src/core/think/prompt.ts` | Main `gbrain think` prompt; answers by reasoning across personal-knowledge evidence and emits strict JSON. |
| Think prompt builders | `src/core/think/prompt.ts` | Builds system prompt, calibration block, user message, and final JSON-only response instruction. |
| Search modality classifier | `src/core/search/llm-intent.ts` | Classifies a search query modality as `text`, `image`, or `both`. |
| Per-chunk synopsis prompt | `src/core/page-summary.ts` | Generates one-sentence chunk synopses for memory and retrieval; prompt version is folded into cache generation. |

### Extraction and dream-cycle prompts

| Prompt | File | Purpose |
| --- | --- | --- |
| Atomic nugget extractor | `src/core/cycle/extract-atoms.ts` | Extracts one to three atomic content nuggets from a transcript as JSON only. |
| Takes extractor | `src/core/cycle/propose-takes.ts` | Extracts gradeable claims/takes from prose and returns only a JSON array. |
| Recurring-pattern prompt | `src/core/cycle/patterns.ts` | Surfaces recurring themes across recent reflections. |
| Conversation worth-processing gate | `src/core/cycle/synthesize.ts` | Decides whether a transcript is worth processing as JSON. |
| Transcript synthesis prompt | `src/core/cycle/synthesize.ts` | Synthesizes a conversation transcript into the personal knowledge brain. |
| Concept synthesis prompt | `src/core/cycle/synthesize-concepts.ts` | Writes a three-to-five-sentence executive summary paragraph for a concept. |
| Calibration profile prompts | `src/core/cycle/calibration-profile.ts` | Summarizes forecaster track record into pattern statements and bias tags. |
| Forecasting-take judge | `src/core/cycle/grade-takes.ts` | Grades single forecasting takes and returns strict JSON. |

### Chat and transcript parsing prompts

| Prompt | File | Purpose |
| --- | --- | --- |
| Regex parse polish prompt | `src/core/conversation-parser/llm-polish.ts` | Polishes regex-parsed chat messages and returns JSON describing operations. |
| Fallback chat parser prompt | `src/core/conversation-parser/llm-fallback.ts` | Parses messages from arbitrary chat-log bodies and returns a JSON array only. |

### Facts, claims, and evaluation prompts

| Prompt | File | Purpose |
| --- | --- | --- |
| Fact classifier | `src/core/facts/classify.ts` | Treats supplied content as data, not instructions, and outputs one JSON object. |
| Fact extractor | `src/core/facts/extract.ts` | Extracts structured facts as a strict single-line JSON object. |
| Contradiction judge | `src/core/eval-contradictions/judge.ts` | Judges contradictions in personal-knowledge chunks, replying JSON only with thresholded contradiction behavior. |
| Takes quality rubric | `src/core/takes-quality-eval/rubric.ts` | Evaluates sampled takes across rubric dimensions. |
| Takes quality runner system | `src/core/takes-quality-eval/runner.ts` | Evaluation judge prompt requiring strict JSON and no markdown fences. |
| Page takes extractor | `src/core/extract-takes-from-pages.ts` | Extracts gradeable claims from longform writing as strict JSON. |
| LongMemEval extractor | `src/eval/longmemeval/extract.ts` | Extracts typed claims/events from chat-session transcripts as a JSON array only. |
| LongMemEval answer prompt | `src/commands/eval-longmemeval.ts` | Answers questions about long-running conversations from retrieved evidence. |
| Notability eval classifier | `src/commands/notability-eval.ts` | Classifies mined paragraphs into high, medium, or low notability tiers. |

### Brainstorming and generation prompts

| Prompt | File | Purpose |
| --- | --- | --- |
| Brainstorm idea generator | `src/core/brainstorm/orchestrator.ts` | Uses bisociation to collide two brain pages and surface non-trivial ideas. |
| Brainstorm idea judge | `src/core/brainstorm/judges.ts` | Scores brainstorm ideas structurally and responds with JSON only. |
| LLM topic-shift chunker | `src/core/chunkers/llm.ts` | Finds the first major topic shift in numbered document segments and responds with a number or `NONE`. |
| Book mirror chapter prompt | `src/commands/book-mirror.ts` | Analyzes a chapter for personalized book-mirror output. |
| Voice gate | `src/core/calibration/voice-gate.ts` | Judges whether a surface should show conversational or academic voice. |

### SkillOpt and skill-evolution prompts

| Prompt | File | Purpose |
| --- | --- | --- |
| Bootstrap benchmark generator | `src/core/skillopt/bootstrap-benchmark.ts` | Generates deterministic rule checks for a user intent that triggers a skill. |
| From-skill benchmark generator | `src/core/skillopt/bootstrap-benchmark.ts` | Infers expected output quality from `SKILL.md` and generates realistic JSONL benchmark tasks. |
| Failure reflector | `src/core/skillopt/reflect.ts` | Analyzes failed agent trajectories and proposes edits to a skill document. |
| Success reflector | `src/core/skillopt/reflect.ts` | Analyzes successful trajectories and proposes edits to make the skill repeat successes. |
| LLM judge | `src/core/skillopt/score.ts` | Scores an agent final output against a rubric and returns strict JSON. |
| Cross-modal evaluator | `src/core/cross-modal-eval/runner.ts` | Strict quality evaluator for task/output scoring across configured dimensions. |

### Subagent prompt

| Prompt | File | Purpose |
| --- | --- | --- |
| Default subagent system prompt | `src/core/minions/system-prompt.ts` | Default identity for GBrain subagents, rendered with optional deterministic tool guidance. |

## 10. Practical shortcuts

- Need to answer a question from the brain? Use `skills/query/SKILL.md`; if it is relationship-oriented, use its graph-query path.
- Need to save arbitrary content? Use `skills/capture/SKILL.md` unless the content type clearly maps to `idea-ingest`, `media-ingest`, `meeting-ingestion`, or `voice-note-ingest`.
- Need to write or update people or company pages? Use `skills/enrich/SKILL.md` and respect brain-first lookup plus quality conventions.
- Need web or current research? Use `skills/perplexity-research/SKILL.md` for freshness deltas and `skills/academic-verify/SKILL.md` for scholarly claims.
- Need background automation? Use `skills/minion-orchestrator/SKILL.md` for durable jobs, not hidden ad-hoc background work.
- Need schema or type changes? Use `skills/schema-author/SKILL.md`; if cleaning up type sprawl, use `skills/schema-unify/SKILL.md`.
- Unsure which route applies? Prefer the most specific matching skill; if still ambiguous, use `skills/ask-user/SKILL.md`.
