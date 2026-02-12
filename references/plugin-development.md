# Plugin Development Reference

## Plugin Interface

```typescript
const plugin: Plugin = {
  name: 'my-plugin',
  description: 'What this plugin does',
  priority: 10,                    // Loading order (lower number = loads first, plugin-sql uses 0)
  dependencies: ['@elizaos/plugin-sql'],
  init: async (config, runtime) => { /* setup — called during registerPlugin() */ },
  actions: [],
  providers: [],
  evaluators: [],
  services: [],
  routes: [],                      // HTTP endpoints
  events: {},                      // Event handlers
  tests: [],
  config: {},
  schema: {},                      // Drizzle ORM table definitions for auto-migration
};
```

### Plugin Init Timing (CRITICAL)

`plugin.init(config, runtime)` is called during `registerPlugin()`, which happens during agent startup. The database adapter may NOT be ready yet (plugin-sql registers it at priority 0).

**Do NOT** in `init()`:
- Call `runtime.createTask()` — adapter not ready → `TypeError: undefined is not an object`
- Call `runtime.createMemory()` — same issue
- Access `runtime.databaseAdapter.db` — may be undefined

**Instead**, defer DB-dependent setup to:
- **Service `start(runtime)`** — called after all plugins are registered and adapters are ready
- **`ProjectAgent.init(runtime)`** — called after full agent initialization
- **TaskWorker `execute()`** — runs after everything is bootstrapped

```typescript
// WRONG — will crash
const plugin: Plugin = {
  init: async (config, runtime) => {
    await runtime.createTask({ name: 'MY_TASK', tags: ['repeat'] }); // ERROR!
  },
};

// RIGHT — defer to service start
class MyService extends Service {
  static async start(runtime: IAgentRuntime) {
    const service = new MyService(runtime);
    await runtime.createTask({ name: 'MY_TASK', tags: ['repeat'] }); // OK here
    return service;
  }
}
```

## Actions

Actions are capabilities — what agents can do. Name them `VERB_NOUN`.

```typescript
const myAction: Action = {
  name: 'SWAP_TOKEN',
  description: 'Swap one token for another on DEX',
  similes: ['exchange token', 'trade token'],
  examples: [
    [
      { name: 'user', content: 'Swap 1 ETH for USDC' },
      { name: 'agent', content: 'Executing swap of 1 ETH for USDC...' },
    ],
  ],
  validate: async (runtime: IAgentRuntime, message: Memory): Promise<boolean> => {
    return message.content?.text?.toLowerCase().includes('swap') ?? false;
  },
  handler: async (
    runtime: IAgentRuntime,
    message: Memory,
    state?: State,
    options?: HandlerOptions,
    callback?: HandlerCallback
  ): Promise<ActionResult> => {
    // Access previous step results in multi-step plans
    const previousResults = state?.data?.actionResults || [];

    // Send progress via callback
    if (callback) {
      await callback({ text: 'Finding best route...', thought: 'Checking DEX aggregators' });
    }

    try {
      const result = await executeSwap(/* ... */);
      return {
        success: true,
        text: `Swapped 1 ETH for ${result.amount} USDC`,
        data: { txHash: result.hash, amount: result.amount }, // Available to next steps
      };
    } catch (error) {
      return { success: false, error: error.message }; // Never throw — return error
    }
  },
};
```

### HandlerOptions (Multi-Step Planning)

```typescript
interface HandlerOptions {
  actionContext?: {
    previousResults: ActionResult[];
    currentStep: number;
    totalSteps: number;
  };
  actionPlan?: {
    totalSteps: number;
    currentStep: number;
    steps: Array<{ action: string; status: 'pending' | 'completed' | 'failed'; result?: ActionResult }>;
    thought: string;
  };
}
```

### Action Best Practices

- Always return `ActionResult` with `success` field
- Never throw from handlers — return `{ success: false, error }` instead
- Use callbacks for progress updates on long operations
- Store structured data in `data` field for downstream actions
- Be idempotent — safe to retry on failure
- Include diverse `examples` for better LLM selection
- Use `similes` for alternative trigger phrases

## Providers

Providers inject context into LLM prompts — agent "senses."

```typescript
const walletProvider: Provider = {
  name: 'WALLET_BALANCE',
  description: 'Current wallet balances across chains',
  dynamic: true,          // Re-fetch every time (not cached)
  position: -50,          // Execute early (lower = earlier, range -100 to 100)
  private: false,         // Include in default provider list
  get: async (runtime: IAgentRuntime, message: Memory, state?: State): Promise<ProviderResult> => {
    try {
      const balances = await fetchBalances(runtime.getSetting('WALLET_ADDRESS'));
      return {
        text: `Wallet: ${balances.total} USD across ${balances.chains.length} chains`,
        values: { totalBalance: balances.total.toString() },
        data: { balances: balances.detailed },
      };
    } catch (error) {
      runtime.logger.error('Wallet provider error:', error);
      return { text: '', values: {}, data: {} }; // Never throw
    }
  },
};
```

### Provider Best Practices

- Return quickly — use timeouts for external calls
- Never throw — return empty `{ text: '', values: {}, data: {} }`
- Use `position` to control execution order (-100 first, 100 last)
- Set `dynamic: true` only when data must be fresh every call
- Keep text concise — don't bloat LLM context window

### Core Providers (Bootstrap)

characterProvider, timeProvider, knowledgeProvider, recentMessagesProvider, actionsProvider, factsProvider, settingsProvider, entitiesProvider, relationshipsProvider, worldProvider, anxietyProvider, attachmentsProvider, capabilitiesProvider, providersProvider, evaluatorsProvider, rolesProvider, choiceProvider, actionStateProvider

## Evaluators

Post-processors that analyze responses and extract information.

```typescript
const sentimentEvaluator: Evaluator = {
  name: 'SENTIMENT_TRACKER',
  description: 'Track conversation sentiment over time',
  similes: ['mood tracker'],
  alwaysRun: false,       // true = run on every response
  examples: [{ prompt: 'User seems frustrated', response: 'Stored negative sentiment' }],
  validate: async (runtime, message) => true,
  handler: async (runtime: IAgentRuntime, message: Memory, state?: State): Promise<any> => {
    const sentiment = await analyzeSentiment(message.content.text);
    if (sentiment) {
      await runtime.memory.create({
        type: 'FACT',
        content: `User sentiment: ${sentiment.label}`,
        metadata: { score: sentiment.score, extractedAt: new Date().toISOString() },
      });
    }
    return { sentiment };
  },
};
```

## Services

Long-running singleton processes for persistent connections.

```typescript
class PriceService extends Service {
  static serviceType = 'price-feed';
  capabilityDescription = 'Real-time token price data';
  private ws: WebSocket | null = null;
  private cache: Map<string, number> = new Map();
  private refreshInterval: NodeJS.Timer | null = null;

  constructor(protected runtime: IAgentRuntime) { super(); }

  static async start(runtime: IAgentRuntime): Promise<PriceService> {
    const service = new PriceService(runtime);
    const apiKey = runtime.getSetting('PRICE_API_KEY');
    if (!apiKey) {
      runtime.logger.warn('PRICE_API_KEY not set, service disabled');
      return service; // Graceful degradation
    }
    await service.connect(apiKey);
    service.refreshInterval = setInterval(() => service.refresh(), 30000);

    // Delayed non-critical init
    setTimeout(async () => {
      try { await service.loadHistoricalData(); }
      catch (e) { runtime.logger.error('Historical data load failed', e); }
    }, 5000);

    return service;
  }

  async stop(): Promise<void> {
    if (this.refreshInterval) clearInterval(this.refreshInterval);
    if (this.ws) this.ws.close();
    this.cache.clear();
  }

  async getPrice(symbol: string): Promise<number | null> {
    return this.cache.get(symbol) ?? null;
  }
}
```

### Predefined Service Types

TRANSCRIPTION, VIDEO, BROWSER, PDF, REMOTE_FILES (S3), WEB_SEARCH, EMAIL, TEE, TASK, WALLET, LP_POOL, TOKEN_DATA, DATABASE_MIGRATION, PLUGIN_MANAGER, PLUGIN_CONFIGURATION, PLUGIN_USER_INTERACTION, MESSAGE_SERVICE

### Service Best Practices

- `getService()` returns singleton — same instance every call
- Handle missing config gracefully (warn, don't crash)
- Implement proper cleanup in `stop()` (clear intervals, close connections)
- Use delayed init for non-critical startup work
- Log lifecycle events for debugging
- Design for failure resilience — circuit breakers, retry with backoff

## Routes (HTTP Endpoints)

```typescript
const routes: Route[] = [
  {
    type: 'GET',
    path: '/api/my-data',
    public: false,
    handler: async (req, res, runtime) => {
      const data = await runtime.getService<MyService>('my-service').getData();
      res.json({ data });
    },
  },
  {
    type: 'POST',
    path: '/api/upload',
    isMultipart: true,
    handler: async (req, res, runtime) => { /* handle file upload */ },
  },
  {
    type: 'STATIC',
    path: '/dashboard',
    filePath: './public',
  },
];
```

## Event Handlers

```typescript
const plugin: Plugin = {
  events: {
    MESSAGE_RECEIVED: [async (runtime, event) => {
      runtime.logger.info(`Message from ${event.entityId}: ${event.content.text}`);
    }],
    ACTION_COMPLETED: [async (runtime, event) => {
      if (event.action === 'SWAP_TOKEN') await notifyUser(event.result);
    }],
    WORLD_JOINED: [async (runtime, event) => { await syncServerData(event); }],
  },
};
```

### Event Types

World: WORLD_JOINED, WORLD_CONNECTED, WORLD_LEFT
Entity: ENTITY_JOINED, ENTITY_LEFT, ENTITY_UPDATED
Room: ROOM_JOINED, ROOM_LEFT, ROOM_UPDATED
Message: MESSAGE_RECEIVED, MESSAGE_SENT, MESSAGE_DELETED, MESSAGE_UPDATED
Voice: VOICE_MESSAGE_RECEIVED, VOICE_MESSAGE_SENT, VOICE_STARTED, VOICE_ENDED
Execution: RUN_STARTED, RUN_COMPLETED, RUN_FAILED, RUN_TIMEOUT
Action: ACTION_STARTED, ACTION_COMPLETED, ACTION_FAILED
Model: MODEL_USED, MODEL_FAILED

## Database Schemas

Define custom tables using Drizzle ORM and export via plugin `schema` property. Tables are auto-created at startup.

```typescript
import { pgTable, uuid, text, timestamp, jsonb, boolean, index } from 'drizzle-orm/pg-core';
import { sql } from 'drizzle-orm';

// Shared table (all agents access) — no agentId
export const trades = pgTable('trades', {
  id: uuid('id').primaryKey().defaultRandom(),
  pair: text('pair').notNull(),
  amount: text('amount').notNull(),
  price: text('price').notNull(),
  txHash: text('tx_hash'),
  metadata: jsonb('metadata'),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
}, (table) => [
  index('idx_trades_pair').on(table.pair),
]);

// Agent-scoped table (include agentId)
export const agentPortfolio = pgTable('agent_portfolio', {
  id: uuid('id').primaryKey().defaultRandom(),
  agentId: uuid('agent_id').notNull(),
  holdings: jsonb('holdings').$type<{ token: string; amount: string }[]>(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow(),
});

// Register in plugin — schema enables auto-migration
export const plugin: Plugin = {
  name: 'trading-plugin',
  schema: { trades, agentPortfolio },
};
```

### Schema Namespacing

Plugin name `@company/my-plugin` → PostgreSQL schema `company_my_plugin`. Tables get prefixed to prevent conflicts with core tables.

### Migration System

- **Dynamic**: No migration files. `DatabaseMigrationService` discovers schemas, introspects existing tables, generates DDL.
- **Additive only**: Creates tables, adds columns, creates indexes. Never drops anything.
- **Dependency resolution**: Foreign key order is automatically resolved.
- **Limitations**: Column type changes require manual SQL. No rollback support.

### Database Access — Repository Pattern

```typescript
import { eq, and } from 'drizzle-orm';

class TradeRepository {
  constructor(private readonly db: ReturnType<typeof drizzle>) {}

  async create(data: { pair: string; amount: string; price: string }) {
    const [trade] = await this.db.insert(trades).values(data).returning();
    return trade;
  }

  async findByPair(pair: string) {
    return await this.db.select().from(trades).where(eq(trades.pair, pair));
  }

  async updateWithTransaction(tradeId: string, updates: Partial<typeof trades.$inferInsert>) {
    return await this.db.transaction(async (tx) => {
      const [updated] = await tx.update(trades).set(updates)
        .where(eq(trades.id, tradeId)).returning();
      return updated;
    });
  }
}

// Usage in action handler
handler: async (runtime, message) => {
  const db = runtime.databaseAdapter.db;
  const repo = new TradeRepository(db);
  const trade = await repo.create({ pair: 'ETH/USDC', amount: '1.0', price: '3200' });
  return { success: true, data: trade };
};
```

### Foreign Keys to Core Tables

```typescript
import { agentTable } from '@elizaos/plugin-sql/schema';

export const myTable = pgTable('my_data', {
  id: uuid('id').primaryKey().defaultRandom(),
  agentId: uuid('agent_id').notNull()
    .references(() => agentTable.id, { onDelete: 'cascade' }),
});
```

### Table Best Practices

- Prefix table names with plugin name (e.g., `trading_trades`, not `trades`)
- Always include `createdAt` and `updatedAt` timestamps with timezone
- Use JSONB for flexible metadata, not catch-all fields
- Index foreign key columns and frequently queried fields
- Tables without `agentId` are shared across all agents
- Handle both camelCase and snake_case in repository mappers

## Plugin Patterns

### Conditional Loading

```typescript
const plugins = [
  '@elizaos/plugin-bootstrap',   // Always required
  '@elizaos/plugin-sql',         // Always required
  ...(process.env.ANTHROPIC_API_KEY ? ['@elizaos/plugin-anthropic'] : []),
  ...(process.env.DISCORD_API_TOKEN ? ['@elizaos/plugin-discord'] : []),
  ...(process.env.SOLANA_PRIVATE_KEY ? ['@elizaos/plugin-solana'] : []),
];
```

### Action Chaining

Return structured `data` from each action. Next actions access via `state?.data?.actionResults`:

```typescript
// Step 1: Search
handler: async (runtime, message) => ({
  success: true, text: 'Found 5 tokens', data: { tokens: ['ETH', 'USDC', ...] }
});

// Step 2: Uses Step 1 results
handler: async (runtime, message, state) => {
  const searchResult = state?.data?.actionResults?.find(r => r.action === 'SEARCH');
  const tokens = searchResult?.data?.tokens || [];
  // ... use tokens
};
```

### Service Integration in Actions

```typescript
handler: async (runtime, message) => {
  const priceService = runtime.getService<PriceService>('price-feed');
  if (!priceService) return { success: false, error: 'Price service unavailable' };
  const price = await priceService.getPrice('ETH');
  return { success: true, text: `ETH: $${price}`, data: { price } };
};
```
