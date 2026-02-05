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

The v2.0.0 branch (194 commits ahead of develop, version `2.0.0-alpha.2`) restructures packages:

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

### Model Types

```
TEXT_SMALL, TEXT_LARGE, TEXT_COMPLETION, TEXT_REASONING_SMALL, TEXT_REASONING_LARGE,
TEXT_EMBEDDING, TEXT_TOKENIZER_ENCODE/DECODE, IMAGE, IMAGE_DESCRIPTION,
TRANSCRIPTION, TEXT_TO_SPEECH, AUDIO, VIDEO, OBJECT_SMALL, OBJECT_LARGE, RESEARCH
```

Usage: `await runtime.useModel(ModelType.TEXT_LARGE, { prompt, temperature: 0.7 })`

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
POSTGRES_URL=postgresql://...       # Or use PGLite (default)

# Server
ELIZA_SERVER_AUTH_TOKEN=            # REST API auth (header: X-API-KEY)
ELIZA_UI_ENABLE=true               # Web dashboard
LOG_LEVEL=info                     # debug for verbose

# Platforms
DISCORD_APPLICATION_ID= / DISCORD_API_TOKEN=
TWITTER_API_KEY= / TWITTER_API_SECRET_KEY= / TWITTER_ACCESS_TOKEN= / TWITTER_ACCESS_TOKEN_SECRET=
TELEGRAM_BOT_TOKEN=
SOLANA_PRIVATE_KEY= / SOLANA_RPC_URL=
EVM_PRIVATE_KEY= / ETHEREUM_PROVIDER_MAINNET=

# Knowledge
LOAD_DOCS_ON_STARTUP=true
CTX_KNOWLEDGE_ENABLED=true         # 50% better retrieval via contextual embeddings
```

## Key Gotchas & Troubleshooting

- **Memory types changed in v2**: Now 5 types (DOCUMENT, FRAGMENT, MESSAGE, DESCRIPTION, CUSTOM) — not the old 7.
- **Anthropic has no embedding model**: Always include OpenAI or Ollama as fallback for embeddings.
- **Concurrent agent init**: ElizaOS has 30-second timeout — use a mutex and 2-5s delays between agent startups.
- **Never throw from handlers**: Always return `{ success: false, error }` from actions; return empty result from providers.
- **Twitter requires OAuth 1.0a**: NOT OAuth 2.0.
- **WebSocket clients must listen to `messageBroadcast`**: NOT `message`. Must emit ROOM_JOINING first.
- **Plugin not loading**: Check export in `src/index.ts`, verify in character `plugins` array, run `bun run build`.
- **Service not found**: Verify `static serviceType` matches `getService()` name, ensure `start()` returns instance.
- **Memory search empty**: Lower threshold (default 0.7), check embeddings generated, verify correct type.
- **Action not triggering**: Check `validate()` logic, review `similes` and `examples`, ensure registered in plugin.
- **Database errors**: Verify `POSTGRES_URL` format, check PGLite fallback with `PGLITE_DATA_DIR`.
- **Naming conflicts**: Prefix plugin tables to avoid colliding with ElizaOS tables (agents, memories, entities, rooms).

## Reference Files

Read these reference files as needed for deeper information:

- **[v2 Architecture](references/v2-architecture.md)** — v2.0.0 branch: restructured packages, Python/Rust SDKs, protobuf schemas, capability tiers, autonomy, ServiceBuilder, message service modes, complete type system (26 files), AgentRuntime class, bootstrap tiers, streaming, working memory. Read when working with v2.0.0 branch code.
- **[Plugin Development](references/plugin-development.md)** — Full Action/Provider/Evaluator/Service interfaces, handler signatures, patterns, schemas, routes, events. Read when writing or debugging plugins.
- **[Platform Integrations](references/platform-integrations.md)** — All platform plugins (Discord, Twitter, Telegram, Farcaster), blockchain (Solana, EVM), LLM providers (OpenAI, Anthropic, Ollama, OpenRouter, Google), Knowledge/RAG, SQL, MCP. Read when configuring integrations.
- **[API Reference](references/api-reference.md)** — Complete REST API endpoints (agents, messaging, sessions, memory, rooms, audio, system), WebSocket events, Socket.IO patterns. Read when building API integrations.
- **[Ecosystem](references/ecosystem.md)** — All 57 GitHub repos: starters, showcase agents, Python toolkit, data tools, infrastructure. Read when exploring the ecosystem or finding starter templates.
