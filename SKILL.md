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
  Covers the v2 stable line (default `develop` branch — currently `2.0.0-beta.0` in package.json,
  latest stable tag `v2.0.3` as of 2026-05-20) with TypeScript runtime, Eliza Cloud deployment
  packages, capability tiers (Basic/Extended/Autonomy), ServiceBuilder API,
  x402 payments, and Dexter SDK integration. Legacy v1 (1.4.4) lives on the `main` branch.
  Last verified against the live repo: 2026-06-08.
---

# ElizaOS Expert Skill

## Question Mode

If this skill was invoked with an argument (e.g., `/elizaos how do I create a service?` or `/elizaos what changed in v2?`), treat the argument as a **question to answer** using the knowledge in this skill and its reference files. Answer the question directly and concisely — do NOT modify any code unless the user's original message also asked for code changes. Read the relevant reference files as needed to provide accurate answers.

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

### Branches & Versions (verified 2026-06-08)

| Branch | package.json | Status | Notes |
|--------|---------|--------|---------|
| `develop` (default) | `2.0.0-beta.0` | Active development — v2 stable line | 49 packages, 116+ plugins, TypeScript |
| `main` | `1.4.4` | Legacy v1, frozen | TypeScript-only, 17 packages |
| `v2-develop` | `1.4.4` | **Stale** — identical to `main`, 3956 commits behind develop | Don't use; migrated into `develop` |

**Latest published tag:** `v2.0.3` (2026-05-20). **v2 is now stable** — `v2.0.0` shipped May 2026 with `v2.0.3` as the current patch. Install via `bun i -g @elizaos/cli`.

**Recommendation:** For new work target the **`develop` branch (v2)** — it's where all development happens. Only fall back to `main` (1.4.4) if you need a frozen TypeScript-only baseline; upstream development of v1 has stopped.

### Monorepo Layout — `develop` branch (verified 2026-06-08)

```
packages/             → 49 packages total
  agent/              → Agent runtime composition layer
  app/                → Tauri desktop wrapper
  app-core/           → Shared app logic
  benchmarks/         → Perf benchmarks
  core/               → Core TypeScript SDK (npm: @elizaos/core)
  docs/               → Mintlify docs source (docs.elizaos.ai)
  elizaos/            → CLI (npm: @elizaos/cli)
  examples/           → Example agents
  homepage/           → elizaos.ai marketing site
  logger/             → Standalone logger package
  native/             → Native plugins/runtime (replaces native-plugins/)
  os/                 → ElizaOS core OS layer
  prompts/            → Standalone prompt templates
  scenario-runner/    → Scenario testing harness
  shared/             → Cross-package shared utilities
  skills/             → Reusable agent skill library
  tui/                → Terminal UI
  ui/                 → React web dashboard
  cloud-api/          → Eliza Cloud API
  cloud-frontend/     → Eliza Cloud frontend
  cloud-infra/        → Eliza Cloud infrastructure
  cloud-routing/      → Eliza Cloud routing
  cloud-sdk/          → Eliza Cloud SDK
  cloud-services/     → Eliza Cloud services
  cloud-shared/       → Eliza Cloud shared utilities
  plugin-host-shim/        → Plugin host shim (cross-runtime)
  plugin-host-shim-android/ → Android plugin shim
  plugin-host-shim-electrobun/ → Electrobun plugin shim
  plugin-host-shim-ios/    → iOS plugin shim
  plugin-worker-runtime/   → Worker runtime for plugins
  ... (+ alberta, browser-bridge-extension, chip, contracts, feed,
         os-homepage, plugin-remote-manifest, plugin-sub-agent-claude-code,
         registry, research/, robot, scripts, security, skills, soc2-verify,
         sweagent, test, training, vault)

plugins/              → 116+ first-party plugins at top level (Discord,
                        Telegram, Twitter, Farcaster, OpenAI, Anthropic,
                        Google, Groq, MCP, local-inference, native mobile
                        plugins, cloud integrations, and many more)
```

**Key facts to internalize:**
- **v2 is now stable.** `v2.0.0` shipped May 2026; `v2.0.3` is the current stable tag. The alpha/beta period ended.
- Plugins are **inside** the monorepo at top-level `plugins/`, not in a separate `elizaos-plugins` org. The old separate-org claim is obsolete.
- The **Python and Rust SDK packages** (`packages/python/`, `packages/rust/`) that existed in earlier alpha builds are **no longer present** in the develop branch as of 2026-06-08. The runtime is now TypeScript-only in the monorepo.
- The monorepo has grown significantly: 49 packages (was 21) and 116+ plugins (was 39), with major additions for Eliza Cloud and native mobile support.

### `v2.0.0` Branch (Separate Experimental)

Distinct from `develop`. Layout:
```
packages/  @schemas/ computeruse/ daemon/ docs/ elizaos/ interop/
           milaidy/ mldy/ prompts/ psyop/ python/ rust/ samantha/
           skills/ sweagent/ tui/ typescript/
```

Adds `computeruse/` (computer-use capabilities), `sweagent/` (SWE-Agent integration), `tui/` (terminal UI), and several named character/agent packages (`milaidy`, `mldy`, `psyop`, `samantha`). Use only if you specifically want to track this branch.

### Key v2 Capabilities (vs legacy v1.4.4 on `main`)

Entity Component System (replaces user system), ServiceBuilder (`createService()`/`defineService()`), formal Event system (30+ EventType enum), Task system (`TaskWorker` + persistent queue), multi-agent orchestration (`ElizaOS` class), model handler registry with priority routing, action chaining (`ActionContext`), working memory, x402 payment types, run tracking, capability tiers, autonomy mode, native mobile plugin shims, Eliza Cloud deployment suite.

For full v2 details, read **[v2 Architecture](references/v2-architecture.md)**.

## Quick Start

```bash
bun install -g @elizaos/cli         # v2 stable; add @alpha only to track bleeding-edge develop
elizaos create my-agent              # Scaffold project
elizaos env edit-local               # Set API keys
elizaos start                        # Run (web UI at localhost:3000)
elizaos dev                          # Dev mode with hot reload
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

### Plugin Schema System (Drizzle ORM)

Plugins define custom tables via Drizzle ORM and export as `schema` property. Migrations are **fully automatic** — no migration files needed.

```typescript
import { pgTable, uuid, text, timestamp, jsonb } from 'drizzle-orm/pg-core';

export const myDataTable = pgTable('my_plugin_data', {
  id: uuid('id').primaryKey().defaultRandom(),
  agentId: uuid('agent_id').notNull(),
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
```

### Model Types & Priority Routing

```
TEXT_SMALL, TEXT_LARGE, TEXT_COMPLETION, TEXT_REASONING_SMALL, TEXT_REASONING_LARGE,
TEXT_EMBEDDING, TEXT_TOKENIZER_ENCODE/DECODE, IMAGE, IMAGE_DESCRIPTION,
TRANSCRIPTION, TEXT_TO_SPEECH, AUDIO, VIDEO, OBJECT_SMALL, OBJECT_LARGE, RESEARCH
```

Usage: `await runtime.useModel(ModelType.TEXT_LARGE, { prompt, temperature: 0.7 })`

Model registration uses **priority-based routing** — higher priority wins.

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

## @elizaos/core API Quick Reference (v2 alpha — develop branch)

These are the **exact** type signatures for the current alpha. Use these when writing plugins.

### Handler Signature

```typescript
type Handler = (
  runtime: IAgentRuntime,
  message: Memory,
  state?: State,                    // OPTIONAL — State | undefined
  options?: HandlerOptions,
  callback?: HandlerCallback,
  responses?: Memory[]
) => Promise<ActionResult | void | undefined>;
```

### Action

```typescript
interface Action {
  name: string;
  description: string;
  similes?: string[];
  examples?: ActionExample[][];
  validate(runtime: IAgentRuntime, message: Memory, state?: State): Promise<boolean>;
  handler: Handler;  // Returns Promise<ActionResult | void | undefined>
}

interface ActionResult { success: boolean; text?: string; error?: string;
  values?: Record<string, any>; data?: Record<string, any>; }
```

### Provider

```typescript
interface Provider {
  name: string;
  description?: string;
  dynamic?: boolean;
  position?: number;
  private?: boolean;
  get(runtime: IAgentRuntime, message: Memory, state?: State): Promise<ProviderResult>;
}

interface ProviderResult {
  text?: string;
  values?: Record<string, unknown>;
  data?: Record<string, unknown>;
}
```

### Evaluator

```typescript
interface Evaluator {
  name: string;
  description: string;
  similes?: string[];
  alwaysRun?: boolean;
  examples?: EvaluatorExample[];     // MUST include, at least empty []
  validate(runtime: IAgentRuntime, message: Memory, state?: State): Promise<boolean>;
  handler: Handler;
}
```

### Service

```typescript
abstract class Service {
  runtime!: IAgentRuntime;
  static serviceType: string;         // e.g. "MY_SERVICE" — a string constant
  capabilityDescription?: string;     // Describe what this service does
  static start(runtime: IAgentRuntime): Promise<Service>;
  stop?(): Promise<void>;
}
```

**CRITICAL**: Do NOT name a property `config` in Service subclasses — it conflicts with `Service.config?: Metadata` in the base class. Use `paymentConfig`, `serviceConfig`, etc.

**Plugin registration**: `services: [MyServiceClass]` — pass the class, NOT `new MyServiceClass()`.

### Runtime

```typescript
runtime.getSetting(key: string): string | boolean | number | null;  // Cast with String(val)
runtime.logger.info(obj: Record<string, unknown>, msg: string);     // Pino-style: object first, string second
runtime.logger.warn(msg: string);                                    // Or just string
runtime.useModel(ModelType.TEXT_SMALL, { prompt: '...' }): Promise<string>;
runtime.getService<T extends Service>(serviceType: string): T | null;
```

### Plugin

```typescript
interface Plugin {
  name: string;
  description?: string;
  priority?: number;            // Lower = loads first (plugin-sql uses 0)
  dependencies?: string[];
  init?(config: Record<string, string>, runtime: IAgentRuntime): Promise<void>;
  actions?: Action[];
  providers?: Provider[];
  evaluators?: Evaluator[];
  services?: (typeof Service)[];  // Pass CLASSES, not instances
  routes?: Route[];
  events?: Record<string, Function[]>;
  schema?: Record<string, any>;  // Drizzle ORM tables for auto-migration
}
```

## Bootstrap Plugin (Required)

### Capability Tiers (v2)

**Basic (default):** Providers: actions, actionState, attachments, capabilities, character, entities, evaluators, providers, recentMessages, time, world. Actions: REPLY, IGNORE, NONE. Services: TaskService, EmbeddingGenerationService.

**Extended (opt-in via ENABLE_EXTENDED_CAPABILITIES):** +Providers: choice, contacts, facts, followUps, knowledge, relationships, role, settings. +Actions: addContact, choice, followRoom, generateImage, muteRoom, sendMessage, updateContact, updateRole, updateSettings, etc. +Evaluators: reflection, relationshipExtraction.

**Autonomy (opt-in via ENABLE_AUTONOMY):** +Providers: adminChat, autonomyStatus. +Actions: sendToAdmin. +Services: AutonomyService.

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

## x402 Payments & Dexter SDK

ElizaOS supports x402 micropayments for agent-to-agent service consumption. The recommended SDK is **@dexterai/x402** (Dexter), which provides:

- **`wrapFetch`** — wraps `fetch` with automatic 402 payment handling
- **`createBudgetAccount`** — autonomous spending controls (total budget, per-request cap, hourly rate limit, domain allowlist)
- **`searchAPIs`** — discover paid APIs on the OpenDexter marketplace
- **Access Pass** — pay once for time-windowed unlimited access
- **Multi-chain** — Base, Polygon, Arbitrum, Optimism, Avalanche, Solana, SKALE

```typescript
import { wrapFetch, createBudgetAccount, searchAPIs } from '@dexterai/x402/client';

// Simple: wrap fetch with auto-pay
const x402Fetch = wrapFetch(fetch, {
  walletPrivateKey: process.env.SOLANA_PRIVATE_KEY,
  evmPrivateKey: process.env.EVM_PRIVATE_KEY,
});

// Agent: budget-controlled fetch
const agent = createBudgetAccount({
  walletPrivateKey: process.env.SOLANA_PRIVATE_KEY,
  budget: { total: '50.00', perRequest: '1.00', perHour: '10.00' },
});
const response = await agent.fetch('https://api.example.com/protected');

// Discovery: search OpenDexter marketplace
const apis = await searchAPIs({ query: 'sentiment analysis', maxPrice: 0.10 });
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
PGLITE_DATA_DIR=/custom/path
ELIZA_DATA_DIR=/custom/data

# Server
SERVER_PORT=3000
SERVER_HOST=0.0.0.0
ELIZA_SERVER_AUTH_TOKEN=
ELIZA_UI_ENABLE=true
LOG_LEVEL=info

# Platforms
DISCORD_APPLICATION_ID= / DISCORD_API_TOKEN=
TWITTER_API_KEY= / TWITTER_API_SECRET_KEY= / TWITTER_ACCESS_TOKEN= / TWITTER_ACCESS_TOKEN_SECRET=
TELEGRAM_BOT_TOKEN=
TELEGRAM_ALLOWED_CHATS=            # JSON array: '["chatId1","chatId2"]'
SOLANA_PRIVATE_KEY= / SOLANA_RPC_URL=
EVM_PRIVATE_KEY= / ETHEREUM_PROVIDER_MAINNET=

# x402 / Dexter
X402_NETWORK_ID=base-sepolia
X402_MAX_AUTO_PAY_USD=0.10
X402_BUDGET_USD=10.00

# Knowledge
LOAD_DOCS_ON_STARTUP=true
CTX_KNOWLEDGE_ENABLED=true
```

## Key Gotchas & Troubleshooting

- **Default embedding dimension is 384**: NOT 1536. Both PGLite and PostgreSQL adapters default to `DIMENSION_MAP[384]`.
- **Anthropic has no embedding model**: Always include OpenAI or Ollama as fallback for embeddings.
- **Plugin init timing**: `plugin.init()` is called during `registerPlugin()`. Do NOT call `runtime.createTask()`, `runtime.createMemory()`, or access `runtime.databaseAdapter` during init. Defer to service `start()` or `ProjectAgent.init()`.
- **plugin-sql priority 0**: Must load first. All other plugins should have higher priority (10+).
- **Schema auto-migration is additive only**: Tables and columns added, never dropped.
- **Never throw from handlers**: Return `{ success: false, error }` from actions; return empty result from providers.
- **Service `config` property**: Do NOT use `config` as a property name in Service subclasses — conflicts with `Service.config?: Metadata` base type. Rename to `paymentConfig`, `serviceConfig`, etc.
- **Plugin `services` field**: Pass the **class** (`[MyService]`), NOT an instance (`[new MyService()]`).
- **Handler `state` is optional**: Type is `State | undefined`. Always use `state?.data?.actionResults` with optional chaining.
- **v2 is stable**: `v2.0.3` is the current stable release (May 2026). Use `bun i -g @elizaos/cli` (no `@alpha` suffix needed). Pin a specific version in production `package.json`.
- **`v2-develop` is dead**: It's identical to `main` (v1.4.4). All v2 work is on `develop` — don't get fooled by the branch name.
- **`runtime.getSetting()` returns `string | boolean | number | null`**: Cast with `String(val)` or null-check before use.
- **Logger is Pino-style**: `runtime.logger.info(obj, message)` — object first, string second. Or just `runtime.logger.info(message)`.
- **Provider `get()` returns `ProviderResult`**: Return `{ text: '...' }`, NOT a plain string.
- **Evaluator `examples` required**: Always include `examples: []` at minimum, or ElizaOS crashes silently.
- **Concurrent agent init**: ElizaOS has 30-second timeout — stagger with 2-5s delays.
- **Twitter requires OAuth 1.0a**: NOT OAuth 2.0.
- **WebSocket clients must listen to `messageBroadcast`**: NOT `message`.
- **TELEGRAM_ALLOWED_CHATS**: Must be JSON stringified array, not comma-separated.

## Reference Files

Read these reference files as needed for deeper information:

- **[v2 Architecture](references/v2-architecture.md)** — v2.0.0 branch: restructured packages, Python/Rust SDKs, protobuf schemas, capability tiers, autonomy, ServiceBuilder, message service modes, complete type system (26 files), AgentRuntime class, bootstrap tiers, streaming, working memory, x402 payment types. Read when working with v2.0.0 branch code.
- **[Plugin Development](references/plugin-development.md)** — Full Action/Provider/Evaluator/Service interfaces, handler signatures, patterns, schemas, routes, events, database access. Read when writing or debugging plugins.
- **[Platform Integrations](references/platform-integrations.md)** — All platform plugins (Discord, Twitter, Telegram, Farcaster), blockchain (Solana, EVM), LLM providers (OpenAI, Anthropic, Ollama, OpenRouter, Google), Knowledge/RAG, SQL, MCP. Read when configuring integrations.
- **[API Reference](references/api-reference.md)** — Complete REST API endpoints (agents, messaging, sessions, memory, rooms, audio, system), WebSocket events, Socket.IO patterns. Read when building API integrations.
- **[Ecosystem](references/ecosystem.md)** — GitHub repos: starters, showcase agents, Python toolkit, data tools, infrastructure. Read when exploring the ecosystem or finding starter templates.
