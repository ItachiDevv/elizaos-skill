---
name: elizaos
description: >
  Expert-level knowledge of the ElizaOS framework and ecosystem for building autonomous AI agents.
  Use when working with ElizaOS projects — creating agents, developing plugins (actions, providers,
  evaluators, services), configuring characters, integrating platforms (Discord, Telegram, Twitter,
  Farcaster), blockchain/DeFi (Solana, EVM), knowledge/RAG systems, memory management, database
  schemas, REST APIs, WebSocket, CLI commands, deployment (Eliza Cloud, Docker, Railway, TEE),
  background tasks, event systems, model providers (OpenAI, Anthropic, Ollama, OpenRouter), or
  any code using @elizaos/core, @elizaos/cli, @elizaos/plugin-*, or the elizaos GitHub repos.
  Also use when discussing AI agent architecture, multi-agent orchestration, or the elizaos ecosystem.
  Covers application integration patterns (embedded runtime vs external server, dynamic context
  injection, agent lifecycle management, dual AI fallback, game/simulation loops, knowledge learning
  systems, memory augmentation with importance scoring and reflection).
  Covers BOTH the stable develop branch AND the v2.0.0 alpha branch with Python/Rust SDKs,
  protobuf schemas, cross-language interop, capability tiers, autonomy system, and ServiceBuilder API.
---

# ElizaOS Expert Skill

## Architecture Overview

ElizaOS is a TypeScript framework (Node.js v23+, Bun) for autonomous AI agents. Core architecture:

```
ElizaOS (multi-agent manager, extends EventTarget)
  └── AgentRuntime instances (implements IAgentRuntime extends IDatabaseAdapter)
       ├── Character     — personality, knowledge, style
       ├── Plugins       — modular capability bundles
       │    ├── Actions     — what agents DO (VERB_NOUN)
       │    ├── Providers   — what agents SEE (context injection)
       │    ├── Evaluators  — what agents LEARN (post-processing)
       │    └── Services    — what agents CONNECT to (singletons)
       ├── Memory        — persistent vector DB (5 types)
       ├── Events        — 30+ async event types
       ├── Models        — provider-agnostic with priority routing
       └── Database      — Drizzle ORM + PostgreSQL/PGLite
```

### Message Processing Pipeline

```
Message In → Store in Memory → Compose State (all Providers) → shouldRespond Decision
  → LLM generates thought + actions + response → Validate & Execute Actions
  → Run Evaluators → Store Response → Deliver via Client
```

Two processing modes: **Single-Shot** (one LLM call) and **Multi-Step** (iterative with accumulated context).

### Monorepo Packages (develop branch)

| Package | Purpose |
|---------|---------|
| `@elizaos/core` | Runtime, types, interfaces, utilities |
| `@elizaos/server` | Express.js backend, REST API, WebSocket |
| `@elizaos/client` | React web dashboard |
| `@elizaos/cli` | CLI tool (`elizaos` command) |
| `@elizaos/app` | Tauri cross-platform desktop app |
| `@elizaos/plugin-bootstrap` | Core message handler (required) |
| `@elizaos/plugin-sql` | Database adapter (required) |

### v2.0.0 Branch (Alpha — Major Restructuring)

The v2.0.0 branch (version `2.0.0-alpha.10` on npm as `@next`) restructures packages:

```
packages/
  @schemas/     → Protobuf schema definitions (cross-language types)
  typescript/   → Core TypeScript package (was @elizaos/core)
  python/       → Python runtime/SDK (NEW)
  rust/         → Rust runtime/SDK (NEW)
  interop/      → Cross-language plugin interop (NEW)
  elizaos/      → CLI (renamed from @elizaos/cli)
  computeruse/  → Computer use capabilities (NEW)
  sweagent/     → SWE Agent (NEW)
  prompts/      → Standalone prompt templates (NEW)
plugins/        → 45+ plugins moved to root-level directory
```

Key v2 additions: Multi-language support, capability tiers (Basic/Extended/Autonomy), ServiceBuilder fluent API, autonomy system, x402 payments, trajectory context, research model type, reasoning models, working memory, form events.

For full v2 details, read **[v2 Architecture](references/v2-architecture.md)**.

## Quick Start

```bash
bun install -g @elizaos/cli
elizaos create my-agent          # Scaffold project
elizaos env edit-local           # Set API keys
elizaos start                    # Run (web UI at localhost:3000)
elizaos dev                      # Dev mode with hot reload
```

## CLI Reference

| Command | Purpose |
|---------|---------|
| `elizaos create [name]` | New project/plugin/agent (`--type project\|plugin\|agent\|tee`) |
| `elizaos start` | Production mode (`--character <paths>`, `-p <port>`) |
| `elizaos dev` | Dev with hot reload (`-b` to build first) |
| `elizaos test` | Run tests (`--type component\|e2e\|all`, `--name <pattern>`) |
| `elizaos deploy` | Deploy to Eliza Cloud |
| `elizaos plugins list\|add\|remove` | Manage plugins |
| `elizaos agent list\|start\|stop\|get\|remove\|clear-memories` | Manage agents |
| `elizaos env list\|edit-local\|reset` | Environment config |
| `elizaos publish` | Publish plugin to registry |
| `elizaos update` | Update CLI and packages |
| `elizaos monorepo` | Clone elizaOS for development |
| `elizaos tee phala <cmd>` | TEE operations (Phala Cloud) |

## Project Structure

```
my-project/
├── src/
│   ├── index.ts           # Entry: exports Project with agents
│   ├── character.ts       # Character definition
│   ├── plugins/           # Custom plugins
│   └── __tests__/         # Tests (Bun test runner)
├── .env                   # API keys (gitignored)
├── package.json
├── tsconfig.json
├── tsup.config.ts
└── .eliza/                # Runtime data, PGLite DB
```

### Entry Point

```typescript
import { type Project, type ProjectAgent } from '@elizaos/core';
import { character } from './character';

const agent: ProjectAgent = {
  character,
  init: async (runtime) => { /* post-init logic */ },
  plugins: [],
};

const project: Project = { agents: [agent] };
export default project;
```

## Character Configuration

```typescript
const character: Character = {
  name: 'TradingBot',
  bio: ['Expert DeFi trader', 'Monitors on-chain activity 24/7'],
  username: 'defi_bot',
  adjectives: ['analytical', 'precise', 'risk-aware'],
  topics: ['DeFi', 'yield farming', 'MEV', 'liquidity'],
  style: {
    all: ['Be concise, data-driven', 'Include numbers when relevant'],
    chat: ['Ask about risk tolerance before suggesting trades'],
    post: ['Include relevant token tickers', 'Keep under 280 chars'],
  },
  messageExamples: [[
    { name: 'user', content: { text: 'Should I buy ETH?' } },
    { name: 'TradingBot', content: { text: 'ETH is at $3,200, up 4.2% today. RSI at 62. What is your time horizon?' } },
  ]],
  knowledge: ['DeFi protocol mechanics', { path: './docs/strategies.md', shared: false }],
  plugins: ['@elizaos/plugin-sql', '@elizaos/plugin-openai', '@elizaos/plugin-solana'],
  settings: {
    model: 'gpt-4o',
    secrets: { OPENAI_API_KEY: process.env.OPENAI_API_KEY },
  },
};
```

## Database Architecture

### Storage Backends

ElizaOS uses **PGLite** (embedded PostgreSQL in Node.js) by default. No external database required for development. For production, set `POSTGRES_URL` to use full PostgreSQL.

```bash
# PGLite (default — no config needed, stores at ./.eliza/.elizadb)
PGLITE_DATA_DIR=/custom/path    # Optional: override PGLite data directory

# PostgreSQL (production)
POSTGRES_URL=postgresql://user:password@host:5432/database
```

Adapter selection via `createDatabaseAdapter()`:
- `POSTGRES_URL` set → `PgDatabaseAdapter` (connection pooling, SSL, retry with backoff)
- No `POSTGRES_URL` → `PgliteDatabaseAdapter` (singleton, file-based, zero config)

Both extend `BaseDrizzleAdapter` and implement `IDatabaseAdapter`.

### Core Tables (auto-created by plugin-sql)

`agents`, `memories`, `entities`, `relationships`, `rooms`, `participants`, `messages`, `embeddings`, `cache`, `logs`, `tasks`

### Embedding Dimensions

**Default: 384** (NOT 1536). Defined via `DIMENSION_MAP`:

```
VECTOR_DIMS: SMALL(384), MEDIUM(512), LARGE(768), XL(1024), XXL(1536), XXXL(3072)
```

Both adapters default to `DIMENSION_MAP[384]`. If using OpenAI text-embedding-3-small (1536 dims), the adapter dimension must be aligned — ElizaOS will auto-detect from the embedding provider plugin.

### Plugin Schema System (Drizzle ORM)

Plugins define custom tables via Drizzle ORM and export as `schema` property. Migrations are **fully automatic** — no migration files needed.

```typescript
import { pgTable, uuid, text, timestamp, jsonb } from 'drizzle-orm/pg-core';

export const myDataTable = pgTable('my_plugin_data', {
  id: uuid('id').primaryKey().defaultRandom(),
  agentId: uuid('agent_id').notNull(),  // Agent-scoped (omit for shared)
  content: text('content').notNull(),
  metadata: jsonb('metadata').default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
});

// Plugin export
export const myPlugin: Plugin = {
  name: 'my-plugin',
  schema: { myDataTable },  // Enables auto-migration
};
```

**Schema namespacing**: Plugin `@company/my-plugin` → PostgreSQL schema `company_my_plugin` → table `company_my_plugin.my_plugin_data`. Prevents naming conflicts.

**Migration stages**: Registration → Schema Discovery → Schema Introspection → Dynamic Migration → Dependency Resolution

**Limitations**: Additive only (new tables, columns, indexes). No drops, no column type changes, no rollbacks. Manual intervention required for breaking schema changes.

### Database Access Pattern

```typescript
// In actions/providers/services — get Drizzle instance
const db = runtime.databaseAdapter.db;

// Use repository pattern (recommended)
class MyRepository {
  constructor(private readonly db: ReturnType<typeof drizzle>) {}
  async findById(id: UUID) {
    return await this.db.select().from(myDataTable).where(eq(myDataTable.id, id));
  }
}

// In action handler
const db = runtime.databaseAdapter.db;
const repo = new MyRepository(db);
const data = await repo.findById(someId);
```

### Foreign Keys to Core Tables

```typescript
import { agentTable } from '@elizaos/plugin-sql/schema';
agentId: uuid('agent_id').notNull()
  .references(() => agentTable.id, { onDelete: 'cascade' })
```

## Key Interfaces

### Memory System

5 types: `DOCUMENT`, `FRAGMENT`, `MESSAGE`, `DESCRIPTION`, `CUSTOM`. Scopes: `shared`, `private`, `room`.

```typescript
// Create memory
await runtime.createMemory({ type: MemoryType.DOCUMENT, content: { text: 'User prefers ETH' },
  metadata: { confidence: 0.9 }, roomId, entityId });

// Semantic search (vector similarity)
const results = await runtime.searchMemories({ type: MemoryType.DOCUMENT,
  query: 'user token preferences', limit: 10, threshold: 0.7 });

// State composition
const state = await runtime.composeState(message);
// Selective: only specific providers
const state = await runtime.composeState(message, ['RECENT_MESSAGES', 'CHARACTER'], true);
```

### Model Types & Priority Routing

```
TEXT_SMALL, TEXT_LARGE, TEXT_COMPLETION, TEXT_REASONING_SMALL, TEXT_REASONING_LARGE,
TEXT_EMBEDDING, TEXT_TOKENIZER_ENCODE/DECODE, IMAGE, IMAGE_DESCRIPTION,
TRANSCRIPTION, TEXT_TO_SPEECH, AUDIO, VIDEO, OBJECT_SMALL, OBJECT_LARGE, RESEARCH
```

Usage: `await runtime.useModel(ModelType.TEXT_LARGE, { prompt, temperature: 0.7 })`

Model registration uses **priority-based routing** — higher priority wins. Multiple plugins can register handlers for the same model type; the highest priority handler is used. Plugin loading order (for LLM): Anthropic → OpenRouter → OpenAI → Google GenAI → Ollama.

### Event Types (30+)

World: WORLD_JOINED/CONNECTED/LEFT | Entity: ENTITY_JOINED/LEFT/UPDATED
Room: ROOM_JOINED/LEFT | Message: MESSAGE_RECEIVED/SENT/DELETED
Voice: VOICE_MESSAGE_RECEIVED/SENT | Run: RUN_STARTED/ENDED/TIMEOUT
Action: ACTION_STARTED/COMPLETED | Evaluator: EVALUATOR_STARTED/COMPLETED
Model: MODEL_USED | Embedding: EMBEDDING_GENERATION_* | Control: CONTROL_MESSAGE
Form: FORM_FIELD_CONFIRMED/CANCELLED

### Service Types

```
transcription, video, browser, pdf, aws_s3, web_search, email, tee, task,
wallet, lp_pool, token_data, message_service, message, post, unknown
```

ServiceTypeRegistry is extensible via TypeScript module augmentation.

## ElizaOS Class (Multi-Agent Manager)

```typescript
import { ElizaOS } from '@elizaos/core';
const elizaOS = new ElizaOS();

const agentIds = await elizaOS.addAgents([
  { character: myCharacter, plugins: ['@elizaos/plugin-sql', '@elizaos/plugin-openai'] },
]);
await elizaOS.startAgents();

// Send messages with callbacks
const result = await elizaOS.handleMessage(agentIds[0], {
  entityId: userId, roomId, content: { text: 'Hello!', source: 'web' },
}, { onResponse: (msg) => console.log(msg), onStreamChunk: (chunk) => process.stdout.write(chunk) });

// Ephemeral/serverless mode
await elizaOS.addAgents([{ character, plugins, databaseAdapter }],
  { ephemeral: true, skipMigrations: true, autoStart: true });

// Health monitoring
const health = await elizaOS.healthCheck();
const keys = await elizaOS.validateApiKeys();
```

## Bootstrap Plugin (Required)

### Capability Tiers (v2)

**Basic (default):** Providers: actions, actionState, attachments, capabilities, character, entities, evaluators, providers, recentMessages, time, world. Actions: REPLY, IGNORE, NONE. Services: TaskService, EmbeddingGenerationService.

**Extended (opt-in via ENABLE_EXTENDED_CAPABILITIES):** +Providers: choice, contacts, facts, followUps, knowledge, relationships, role, settings. +Actions: addContact, choice, followRoom, generateImage, muteRoom, sendMessage, updateContact, updateRole, updateSettings, etc. +Evaluators: reflection, relationshipExtraction.

**Autonomy (opt-in via ENABLE_AUTONOMY):** +Providers: adminChat, autonomyStatus. +Actions: sendToAdmin. +Services: AutonomyService.

Config: `SHOULD_RESPOND_BYPASS_TYPES`, `SHOULD_RESPOND_BYPASS_SOURCES`, `CONVERSATION_LENGTH`, `RESPONSE_TIMEOUT`.

## Background Tasks

```typescript
const worker: TaskWorker = {
  name: 'PRICE_ALERT',
  validate: async (runtime, task) => !!task.metadata?.token,
  execute: async (runtime, options, task) => {
    const price = await checkPrice(task.metadata.token);
    if (price > task.metadata.threshold) await notify(runtime, task);
  },
};

await runtime.createTask({
  name: 'PRICE_ALERT',
  metadata: { token: 'ETH', threshold: 4000, updateInterval: 60000 },
  tags: ['repeat'],  // 'queue' = one-time, 'repeat' = recurring, 'immediate' = run now
});
```

## Deployment

### Eliza Cloud
```bash
elizaos login && elizaos deploy --project-name my-agent
```

### Docker
```dockerfile
FROM oven/bun:1
WORKDIR /app
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile
COPY . .
RUN bun run build
EXPOSE 3000
CMD ["bun", "run", "start"]
```

## Common Environment Variables

```env
# Required
NODE_ENV=development
OPENAI_API_KEY=sk-...              # Or ANTHROPIC_API_KEY
POSTGRES_URL=postgresql://...       # Or use PGLite (default — no config needed)

# Database
PGLITE_DATA_DIR=/custom/path       # Override PGLite storage (default: ./.eliza/.elizadb)
ELIZA_DATA_DIR=/custom/data        # General data directory

# Server
SERVER_PORT=3000                   # Server port (default 3000)
SERVER_HOST=0.0.0.0                # Server bind address
ELIZA_SERVER_AUTH_TOKEN=            # REST API auth (header: X-API-KEY)
ELIZA_UI_ENABLE=true               # Web dashboard
LOG_LEVEL=info                     # debug for verbose
IGNORE_BOOTSTRAP=false             # Skip bootstrap plugin if true

# Platforms
DISCORD_APPLICATION_ID= / DISCORD_API_TOKEN=
TWITTER_API_KEY= / TWITTER_API_SECRET_KEY= / TWITTER_ACCESS_TOKEN= / TWITTER_ACCESS_TOKEN_SECRET=
TELEGRAM_BOT_TOKEN=
TELEGRAM_ALLOWED_CHATS=            # JSON array: '["chatId1","chatId2"]'
SOLANA_PRIVATE_KEY= / SOLANA_RPC_URL=
EVM_PRIVATE_KEY= / ETHEREUM_PROVIDER_MAINNET=

# Knowledge
LOAD_DOCS_ON_STARTUP=true
CTX_KNOWLEDGE_ENABLED=true         # 50% better retrieval via contextual embeddings
```

## Application Integration Patterns

ElizaOS can be integrated into applications two ways:

### Embedded Runtime (Direct Import)
Import `@elizaos/core` directly and manage `AgentRuntime` instances in-process. Best for monorepos.

```typescript
const runtime = new AgentRuntime({ character, plugins, databaseAdapter });
await runtime.initialize();
const response = await runtime.processMessage(/* ... */);
```

### External Server (REST API)
Run ElizaOS standalone (`elizaos start`) and communicate via REST. Best for decoupled architectures.

```typescript
// Create agent
await fetch(`${ELIZA_SERVER}/api/agents`, { method: 'POST', body: JSON.stringify(character) });
// Send message via DM channel
await fetch(`${ELIZA_SERVER}/api/messaging/dm/${agentId}/${userId}`, { method: 'POST' });
await fetch(`${ELIZA_SERVER}/api/messaging/channels/${channelId}/messages`, {
  method: 'POST', body: JSON.stringify({ content: { text }, senderId: userId }),
});
```

### Dynamic Context Injection
Inject game/app state into prompts so agents respond contextually:

```typescript
// Option A: Pre-prompt injection (works with any approach)
const context = `Token balance: ${pet.neoTokens}. Known topics: ${topics.join(', ')}`;
await runtime.processMessage(message, { dynamicContext: context });

// Option B: Custom Provider (ElizaOS native — embedded only)
const gameStateProvider: Provider = {
  name: 'GAME_STATE', dynamic: true, position: -90,
  get: async (runtime, message) => ({
    text: `Player balance: ${balance} tokens\nLocation: ${location}`,
    values: {}, data: {},
  }),
};
```

### Agent Lifecycle
- **Lazy start**: Create agents on first chat, not on app startup
- **Auto-stop**: Stop agents after inactivity (e.g., 30min) to save memory (~50-100MB per agent)
- **Config hot-reload**: Update DB config → stop agent → next message lazy-restarts with new config
- **Knowledge learning**: Add entries to `character.knowledge[]` in DB, stop agent, restart picks them up
- **Stagger startup**: 2-5s delays between concurrent agent inits (30s timeout limit)

### Dual AI Fallback
If ElizaOS availability isn't guaranteed, fall back to direct LLM calls with character context:

```typescript
try {
  return await elizaRuntime.processMessage(message);
} catch {
  const systemPrompt = buildPromptFromCharacter(character);
  return await anthropic.messages.create({ system: systemPrompt, messages: [{ role: 'user', content: message }] });
}
```

For full patterns (memory augmentation, game simulation, multi-agent conversations, deployment), see **[Integration Patterns](references/integration-patterns.md)**.

## Key Gotchas & Troubleshooting

- **Default embedding dimension is 384**: NOT 1536. Both PGLite and PostgreSQL adapters default to `DIMENSION_MAP[384]`. OpenAI text-embedding-3-small produces 1536 dims — ElizaOS auto-detects from the provider plugin, but custom code must match.
- **Anthropic has no embedding model**: Always include OpenAI or Ollama as fallback for embeddings. Without an embedding provider, memory search and knowledge features will fail.
- **Plugin init timing**: `plugin.init(config, runtime)` is called during `registerPlugin()`. The database adapter (from plugin-sql with `priority: 0`) may NOT be fully ready when other plugins init. Do NOT call `runtime.createTask()`, `runtime.createMemory()`, or access `runtime.databaseAdapter` during plugin `init()`. Defer to service `start()` or `ProjectAgent.init()` instead.
- **plugin-sql priority 0**: This plugin must load first. It calls `runtime.registerDatabaseAdapter(dbAdapter)` in its init. All other plugins that need DB access should have higher priority numbers (default: 10+).
- **Schema auto-migration is additive only**: Tables and columns are added automatically, but never dropped. Column type changes require manual SQL migration.
- **Memory types**: 5 types on develop branch (DOCUMENT, FRAGMENT, MESSAGE, DESCRIPTION, CUSTOM). V2 has same 5.
- **Never throw from handlers**: Always return `{ success: false, error }` from actions; return empty result from providers.
- **Concurrent agent init**: ElizaOS has 30-second timeout — use a mutex and 2-5s delays between agent startups.
- **Twitter requires OAuth 1.0a**: NOT OAuth 2.0.
- **WebSocket clients must listen to `messageBroadcast`**: NOT `message`. Must emit ROOM_JOINING first.
- **Plugin not loading**: Check export in `src/index.ts`, verify in character `plugins` array, run `bun run build`.
- **Service not found**: Verify `static serviceType` matches `getService()` name, ensure `start()` returns instance.
- **Memory search empty**: Lower threshold (default 0.7), check embeddings generated, verify correct type.
- **Action not triggering**: Check `validate()` logic, review `similes` and `examples`, ensure registered in plugin.
- **Database errors**: Verify `POSTGRES_URL` format. For PGLite, check `.eliza/.elizadb` directory is writable. Delete `.eliza/` to reset local DB.
- **Naming conflicts**: Prefix plugin tables to avoid colliding with ElizaOS core tables (agents, memories, entities, rooms, participants, messages, embeddings, cache, logs, tasks).
- **Plugin schema export**: Tables must be exported in the plugin's `schema` property to trigger auto-migration. Missing `schema` = no tables created.
- **Database access**: Always use `runtime.databaseAdapter.db` (Drizzle instance), NOT direct Supabase client, for tables managed by ElizaOS. Custom external databases (Supabase REST API) can coexist alongside.
- **TELEGRAM_ALLOWED_CHATS**: Must be JSON stringified array, not comma-separated.
- **Serverless deployment**: ElizaOS agents need persistent processes. Vercel/Netlify Functions will NOT work. Use Railway, Render, Fly.io, or similar.
- **ElizaOS auto-migration vs pre-created indexes**: If you pre-create indexes that ElizaOS expects to create, migration will fail. Only create tables with `IF NOT EXISTS` and let ElizaOS handle its own indexes.
- **Knowledge hot-reload**: Character `knowledge` is loaded at agent startup. To add knowledge at runtime, update the config in your DB then stop the agent — next message will lazy-restart with new knowledge.
- **Agent memory per instance**: Each running ElizaOS agent uses ~50-100MB. Scale accordingly and implement auto-stop for inactive agents.

## Reference Files

Read these reference files as needed for deeper information:

- **[v2 Architecture](references/v2-architecture.md)** — v2.0.0 branch: restructured packages, Python/Rust SDKs, protobuf schemas, capability tiers, autonomy, ServiceBuilder, message service modes, complete type system (26 files), AgentRuntime class, bootstrap tiers, streaming, working memory. Read when working with v2.0.0 branch code.
- **[Plugin Development](references/plugin-development.md)** — Full Action/Provider/Evaluator/Service interfaces, handler signatures, patterns, schemas, routes, events. Read when writing or debugging plugins.
- **[Platform Integrations](references/platform-integrations.md)** — All platform plugins (Discord, Twitter, Telegram, Farcaster), blockchain (Solana, EVM), LLM providers (OpenAI, Anthropic, Ollama, OpenRouter, Google), Knowledge/RAG, SQL, MCP. Read when configuring integrations.
- **[API Reference](references/api-reference.md)** — Complete REST API endpoints (agents, messaging, sessions, memory, rooms, audio, system), WebSocket events, Socket.IO patterns. Read when building API integrations.
- **[Integration Patterns](references/integration-patterns.md)** — Embedded vs external server, dynamic context injection, agent lifecycle (lazy start/auto-stop/config hot-reload), dual AI fallback, game simulation loops, multi-agent conversations, knowledge learning, memory augmentation (importance scoring, reflection, recency decay), token economies, deployment patterns. Read when integrating ElizaOS into applications or games.
- **[Ecosystem](references/ecosystem.md)** — All 57 GitHub repos: starters, showcase agents, Python toolkit, data tools, infrastructure. Read when exploring the ecosystem or finding starter templates.
