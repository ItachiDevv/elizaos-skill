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
---

# ElizaOS Expert Skill

## Architecture Overview

ElizaOS is a TypeScript framework (Node.js v23+, Bun) for autonomous AI agents. Core architecture:

```
ElizaOS (multi-agent manager)
  └── AgentRuntime instances (one per agent)
       ├── Character     — personality, knowledge, style
       ├── Plugins       — modular capability bundles
       │    ├── Actions     — what agents DO (VERB_NOUN)
       │    ├── Providers   — what agents SEE (context injection)
       │    ├── Evaluators  — what agents LEARN (post-processing)
       │    └── Services    — what agents CONNECT to (singletons)
       ├── Memory        — persistent vector DB (7 types)
       ├── Events        — async event-driven reactions
       └── Database      — Drizzle ORM + PostgreSQL/PGLite
```

### Message Processing Pipeline

```
Message In → Store in Memory → Compose State (all Providers) → LLM decides Action
  → Validate & Execute Action → Run Evaluators → Store Response → Deliver via Client
```

### Monorepo Packages

| Package | Purpose |
|---------|---------|
| `@elizaos/core` | Runtime, types, interfaces, utilities |
| `@elizaos/server` | Express.js backend, REST API, WebSocket |
| `@elizaos/client` | React web dashboard |
| `@elizaos/cli` | CLI tool (`elizaos` command) |
| `@elizaos/app` | Tauri cross-platform desktop app |
| `@elizaos/plugin-bootstrap` | Core message handler (required) |
| `@elizaos/plugin-sql` | Database adapter (required) |

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
| `elizaos login` | Auth with Eliza Cloud |
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
    { name: 'user', content: 'Should I buy ETH?' },
    { name: 'TradingBot', content: 'ETH is at $3,200, up 4.2% today. RSI at 62 — not overbought. What is your time horizon and risk tolerance?' },
  ]],
  knowledge: ['DeFi protocol mechanics', { path: './docs/strategies.md', shared: false }],
  plugins: ['@elizaos/plugin-sql', '@elizaos/plugin-openai', '@elizaos/plugin-solana'],
  settings: {
    model: 'gpt-4o',
    secrets: { OPENAI_API_KEY: process.env.OPENAI_API_KEY },
  },
};
```

## ElizaOS Class (Multi-Agent Manager)

```typescript
import { ElizaOS } from '@elizaos/core';

const elizaOS = new ElizaOS();

// Add and start agents
const agentIds = await elizaOS.addAgents([
  { character: myCharacter, plugins: ['@elizaos/plugin-sql', '@elizaos/plugin-openai'] },
]);
await elizaOS.startAgents();

// Send messages
const result = await elizaOS.handleMessage(agentIds[0], {
  entityId: userId,
  roomId: roomId,
  content: { text: 'Hello!', source: 'web' },
});

// Multi-agent parallel messaging
await elizaOS.handleMessages([
  { agentId: agent1, message: msg1 },
  { agentId: agent2, message: msg2 },
]);

// Agent discovery
elizaOS.getAgent(id);
elizaOS.getAgentByName('TradingBot');
elizaOS.getAgents();

// Health monitoring
const health = await elizaOS.healthCheck();        // Map<UUID, {alive, responsive, ...}>
const keys = await elizaOS.validateApiKeys();      // Map<UUID, boolean>

// Ephemeral/serverless mode (skip registry, no persistence)
await elizaOS.addAgents([{ character, plugins, databaseAdapter }],
  { ephemeral: true, skipMigrations: true, autoStart: true });
```

## Memory System

7 types: `MESSAGE`, `FACT`, `DOCUMENT`, `RELATIONSHIP`, `GOAL`, `TASK`, `ACTION`.

```typescript
// Create
await runtime.memory.create({
  type: 'FACT', content: 'User prefers ETH over BTC',
  metadata: { confidence: 0.9, category: 'preference' },
});

// Semantic search (vector similarity)
const results = await runtime.memory.search({
  type: 'FACT', query: 'user token preferences', limit: 10, threshold: 0.7,
});

// Filtered query
const tasks = await runtime.memory.query({
  type: 'TASK', filter: { 'metadata.status': 'pending' },
});

// Update / Delete
await runtime.memory.update(id, { metadata: { status: 'completed' } });
await runtime.memory.delete(id);
```

### State Composition

```typescript
const state = await runtime.composeState(message);
// Selective: only specific providers
const state = await runtime.composeState(message, ['RECENT_MESSAGES', 'CHARACTER'], true);
```

## Bootstrap Plugin (Required)

13 actions: REPLY, IGNORE, NONE, FOLLOW_ROOM, UNFOLLOW_ROOM, MUTE_ROOM, UNMUTE_ROOM, SEND_MESSAGE, UPDATE_CONTACT, CHOOSE_OPTION, UPDATE_ROLE, UPDATE_SETTINGS, GENERATE_IMAGE.

17 providers: character, time, knowledge, recentMessages, actions, facts, settings, entities, relationships, world, anxiety, attachments, capabilities, providers, evaluators, roles, choice, actionState.

Reflection evaluator: Extracts facts, tracks relationships, builds knowledge after each interaction.

Config: `SHOULD_RESPOND_BYPASS_TYPES`, `SHOULD_RESPOND_BYPASS_SOURCES`, `CONVERSATION_LENGTH`, `RESPONSE_TIMEOUT`.

## Background Tasks

```typescript
// Task worker definition
const worker: TaskWorker = {
  name: 'PRICE_ALERT',
  validate: async (runtime, task) => !!task.metadata?.token,
  execute: async (runtime, options, task) => {
    const price = await checkPrice(task.metadata.token);
    if (price > task.metadata.threshold) await notify(runtime, task);
    await runtime.updateTask(task.id, { metadata: { ...task.metadata, lastCheck: Date.now() } });
  },
};

// Create recurring task
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
elizaos containers list          # View deployments
elizaos containers logs <id>     # Tail logs
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

### TEE (Trusted Execution Environment)
```bash
elizaos create --type tee my-tee-agent
# Set TEE_MODE=PRODUCTION, TEE_VENDOR=phala, WALLET_SECRET_SALT
elizaos tee phala auth login <api-key>
elizaos tee phala cvms create --name my-agent --compose ./docker-compose.yml
```

## Common Environment Variables

```env
# Required
NODE_ENV=development
OPENAI_API_KEY=sk-...              # Or ANTHROPIC_API_KEY
POSTGRES_URL=postgresql://...       # Or use PGLite (default)

# Server
ELIZA_SERVER_AUTH_TOKEN=            # REST API auth
ELIZA_UI_ENABLE=true               # Web dashboard
LOG_LEVEL=info                     # debug for verbose

# Optional platforms
DISCORD_APPLICATION_ID= / DISCORD_API_TOKEN=
TWITTER_API_KEY= / TWITTER_API_SECRET_KEY= / TWITTER_ACCESS_TOKEN= / TWITTER_ACCESS_TOKEN_SECRET=
TELEGRAM_BOT_TOKEN=
SOLANA_PRIVATE_KEY= / SOLANA_RPC_URL=
EVM_PRIVATE_KEY= / ETHEREUM_PROVIDER_MAINNET=
```

## Key Gotchas & Troubleshooting

- **Concurrent agent init**: ElizaOS has 30-second timeout — use a mutex and 2-5s delays between agent startups. Serialize recovery on restart.
- **Anthropic embeddings**: Anthropic has no embedding model — always include OpenAI or Ollama as fallback for embeddings.
- **Plugin not loading**: Check export in `src/index.ts`, verify in character `plugins` array, run `bun run build`.
- **Service not found**: Verify `static serviceType` matches `getService()` name, ensure `start()` returns instance.
- **Memory search empty**: Lower threshold (default 0.7), check embeddings generated, verify correct type.
- **Action not triggering**: Check `validate()` logic, review `similes` and `examples`, ensure registered in plugin.
- **Database errors**: Verify `POSTGRES_URL` format, check PGLite fallback with `PGLITE_DATA_DIR`.
- **Naming conflicts**: Prefix platform tables to avoid colliding with ElizaOS tables (agents, memories, entities, rooms).
- **Never throw from handlers**: Always return `{ success: false, error }` from actions; return empty result from providers.

## Reference Files

For deeper information, read these reference files as needed:

- **[Plugin Development](references/plugin-development.md)** — Full Action/Provider/Evaluator/Service interfaces, handler signatures, patterns, schemas, routes, events. Read when writing or debugging plugins.
- **[Platform Integrations](references/platform-integrations.md)** — All platform plugins (Discord, Twitter, Telegram, Farcaster), blockchain (Solana, EVM), LLM providers (OpenAI, Anthropic, Ollama, OpenRouter, Google), Knowledge/RAG, SQL, MCP. Read when configuring integrations.
- **[API Reference](references/api-reference.md)** — Complete REST API endpoints (agents, messaging, sessions, memory, rooms, audio, system), WebSocket events, Socket.IO patterns. Read when building API integrations.
- **[Ecosystem](references/ecosystem.md)** — All 57 GitHub repos: starters, showcase agents, Python toolkit, data tools, infrastructure. Read when exploring the ecosystem or finding starter templates.
