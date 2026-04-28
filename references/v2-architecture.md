# ElizaOS v2 Architecture Reference

Branch: **`develop`** (default branch — v2 alpha lives here, NOT `v2-develop` which is stale v1).
Root version: **`2.0.0-alpha.176`** in `develop/package.json`; latest published tag **`v2.0.0-alpha.442`** (2026-04-27).
Verified against the live repo: 2026-04-28. Node 23+, Bun 1.3.4+.

## Package Layout (verified)

The monorepo is **polyglot** (TypeScript + Python + Rust) with first-party plugins inside the repo at top-level `plugins/`. Plugins are no longer in a separate org — the old `elizaos-plugins/` organization claim is obsolete.

### Monorepo Packages (`packages/` — 21 entries on `develop`)

| Package dir | Purpose |
|---------|---------|
| `agent/` | Agent runtime composition layer (wires runtime + plugins) |
| `app/` | Tauri desktop wrapper |
| `app-core/` | Shared application logic |
| `benchmarks/` | Performance benchmarks |
| `docs/` | Mintlify docs source (docs.elizaos.ai) |
| `elizaos/` | CLI (publishes to npm as `@elizaos/cli`) |
| `examples/` | Example agent projects |
| `homepage/` | elizaos.ai marketing site |
| `interop/` | Cross-language plugin interoperability |
| `native-plugins/` | Built-in core plugins (sql, bootstrap, etc.) |
| `prompts/` | Standalone prompt templates |
| `python/` | **Python runtime + SDK** — `pyproject.toml`, uv-managed, full `elizaos` package |
| `rust/` | **Rust runtime + SDK** — `Cargo.toml`, builds native + WASM (`build-wasm.sh`, `pkg-node/`) |
| `scenario-runner/` | Scenario testing harness |
| `scenario-schema/` | Scenario format schema |
| `schemas/` | **Protobuf schemas** — `buf.yaml` + `eliza/v1/*.proto` (cross-language wire format) |
| `shared/` | Cross-package utilities |
| `skills/` | Reusable agent skill library |
| `templates/` | Project templates for `elizaos create` |
| `typescript/` | Core TypeScript SDK (publishes as `@elizaos/core`) |
| `ui/` | React web dashboard |

### Top-level `plugins/` directory (39 entries on `develop`)

First-party plugins live inside the monorepo, not a separate org. Includes platform integrations (Discord, Telegram, Twitter, Farcaster), LLM providers (OpenAI, Anthropic, Ollama, OpenRouter, Google), chains (Solana, EVM), knowledge/RAG, MCP, and more.

### `v2.0.0` Branch (separate experimental — different layout)

For reference, the `v2.0.0` branch (not `develop`) has a different package set: `@schemas/`, `computeruse/`, `daemon/`, `elizaos/`, `interop/`, `milaidy/`, `mldy/`, `prompts/`, `psyop/`, `python/`, `rust/`, `samantha/`, `skills/`, `sweagent/`, `tui/`, `typescript/`. Adds computer-use, SWE-Agent integration, terminal UI, and a handful of named character/agent packages. Most users should track `develop`, not `v2.0.0`.

### `main` Branch (legacy v1.4.4)

TypeScript-only, frozen. 17 packages: `api-client, app, cli, client, config, core, elizaos, plugin-bootstrap, plugin-dummy-services, plugin-quick-starter, plugin-sql, plugin-starter, project-starter, project-tee-starter, server, service-interfaces, test-utils`. Use only if you need a frozen v1 baseline.

## Type System

All types in `packages/core/src/types/` as separate files re-exported from `types/index.ts`.

### Plugin Interface (`types/plugin.ts`)

```typescript
interface Plugin {
  name: string;                           // REQUIRED
  description: string;                    // REQUIRED
  init?: (config: Record<string, string>, runtime: IAgentRuntime) => Promise<void>;
  config?: Record<string, unknown>;       // Values stringified and passed to init()
  services?: (typeof Service)[];          // Pass class, NOT instance
  componentTypes?: ComponentType[];       // Entity Component System types
  actions?: Action[];
  providers?: Provider[];
  evaluators?: Evaluator[];
  adapter?: IDatabaseAdapter;             // Custom DB adapter
  models?: Record<string, ModelHandler>;  // Model handlers by ModelType
  events?: PluginEvents;                  // Event handlers map
  routes?: Route[];                       // HTTP endpoints
  tests?: TestSuite[];
  dependencies?: string[];               // Auto-resolved with topological sort
  testDependencies?: string[];
  priority?: number;                      // Higher = model handlers preferred
  schema?: Record<string, unknown>;       // Zod schema for env validation / Drizzle tables
}
```

Validation (`isValidPluginShape`) requires `name` plus at least one of: `init`, `services`, `providers`, `actions`, `evaluators`, or `description`.

### Project & Agent

```typescript
interface Project { agents: ProjectAgent[]; }
interface ProjectAgent {
  character: Character;
  init?: (runtime: IAgentRuntime) => Promise<void>;
  plugins?: (string | Plugin)[];     // String names auto-resolved
  tests?: TestSuite | TestSuite[];
}
```

### Action / Handler

```typescript
type Handler = (
  runtime: IAgentRuntime,
  message: Memory,
  state?: State,
  options?: HandlerOptions,
  callback?: HandlerCallback,
  responses?: Memory[]
) => Promise<ActionResult | void | undefined>;

interface Action {
  name: string; description: string; similes?: string[];
  examples?: ActionExample[][]; suppressInitialMessage?: boolean;
  validate: Validator; handler: Handler;
  [key: string]: unknown;  // Extensible
}

interface ActionResult {
  success: boolean;        // REQUIRED
  text?: string; error?: string;
  values?: Record<string, unknown>; data?: Record<string, unknown>;
}

interface HandlerOptions {
  actionContext?: { previousResults: ActionResult[]; currentStep: number; totalSteps: number; };
  actionPlan?: ActionPlan;
  [key: string]: unknown;
}
```

### Provider

```typescript
interface Provider {
  name: string; description?: string;
  dynamic?: boolean;     // Excluded from default composeState
  position?: number;     // Lower = earlier (-100 to 100)
  private?: boolean;     // Must be called explicitly
  get(runtime: IAgentRuntime, message: Memory, state: State): Promise<ProviderResult>;
}

interface ProviderResult {
  text?: string; values?: Record<string, unknown>; data?: Record<string, unknown>;
}
```

### Evaluator

```typescript
interface Evaluator {
  name: string; description: string; similes?: string[];
  examples: EvaluationExample[];   // REQUIRED (not optional)
  validate: Validator; handler: Handler;
  alwaysRun?: boolean;             // Skips validate() if true
}
```

### Service

```typescript
abstract class Service {
  protected runtime?: IAgentRuntime;
  abstract stop(): Promise<void>;
  abstract capabilityDescription: string;
  static serviceType: string;
  config?: Record<string, unknown>;
  static start?(runtime: IAgentRuntime): Promise<Service>;
  static stop?(): Promise<void>;
  static registerSendHandlers?(runtime: IAgentRuntime): Promise<void>;
}
```

ServiceBuilder alternatives (in `packages/core/src/services.ts`):
```typescript
// Fluent builder
const MyService = createService<Service>('my-service')
  .withDescription('Does something')
  .withStart(async (runtime) => { /* return service instance */ })
  .withStop(async () => { /* cleanup */ })
  .build();

// Declarative
const MyService = defineService({
  serviceType: 'my-service',
  description: 'Does something',
  start: async (runtime) => { /* return service instance */ },
  stop: async () => { /* cleanup */ }
});
```

ServiceType extensible via module augmentation on `ServiceTypeRegistry`.

Services retrieved via:
```typescript
runtime.getService<T>(type);           // First match
runtime.getServicesByType<T>(type);    // All matches
runtime.hasService(type);              // Boolean check
await runtime.getServiceLoadPromise(type); // Wait until ready
```

### State

```typescript
interface State {
  values: Record<string, string>;  // Flat KV from providers
  data: StateData;                 // Structured data cache
  text: string;                    // Concatenated provider text
  [key: string]: unknown;
}

interface StateData {
  room?: Room; world?: World; entity?: Entity;
  providers?: Record<string, ProviderResult>;
  actionPlan?: ActionPlan; actionResults?: ActionResult[];
  workingMemory?: Record<string, WorkingMemoryEntry>;
  [key: string]: unknown;
}
```

### Memory

```typescript
enum MemoryType { DOCUMENT, FRAGMENT, MESSAGE, DESCRIPTION, CUSTOM }
type MemoryScope = 'shared' | 'private' | 'room';

interface Memory {
  id?: UUID; entityId?: UUID; agentId?: UUID;
  roomId?: UUID; worldId?: UUID;
  content: Content; embedding?: number[];
  unique?: boolean; similarity?: number;
  metadata?: MemoryMetadata;
}
```

**Changes from v1.4.x:** `userId` → `entityId`, `worldId` added, typed MemoryType enum, MemoryScope for visibility.

### Model System

```typescript
const ModelType = {
  TEXT_SMALL, TEXT_LARGE, TEXT_COMPLETION,
  TEXT_REASONING_SMALL, TEXT_REASONING_LARGE,
  TEXT_EMBEDDING, TEXT_TOKENIZER_ENCODE, TEXT_TOKENIZER_DECODE,
  IMAGE, IMAGE_DESCRIPTION, TRANSCRIPTION, TEXT_TO_SPEECH,
  AUDIO, VIDEO, OBJECT_SMALL, OBJECT_LARGE,
} as const;
```

Plugins register model handlers:
```typescript
runtime.registerModel(ModelType.TEXT_LARGE, handler, 'openai', 10);
```

Multiple handlers per type; highest priority wins with provider fallback.

Usage:
```typescript
const text = await runtime.useModel(ModelType.TEXT_LARGE, { prompt, temperature: 0.7 });
const text = await runtime.useModel(ModelType.TEXT_LARGE, { prompt, onStreamChunk: (chunk) => {} });
const obj = await runtime.useModel(ModelType.OBJECT_SMALL, { prompt, schema, output: 'object' });
```

## Event System

```typescript
enum EventType {
  WORLD_JOINED, WORLD_CONNECTED, WORLD_LEFT,
  ENTITY_JOINED, ENTITY_LEFT, ENTITY_UPDATED,
  ROOM_JOINED, ROOM_LEFT,
  MESSAGE_RECEIVED, MESSAGE_SENT, MESSAGE_DELETED,
  CHANNEL_CLEARED,
  VOICE_MESSAGE_RECEIVED, VOICE_MESSAGE_SENT,
  REACTION_RECEIVED, POST_GENERATED, INTERACTION_RECEIVED,
  RUN_STARTED, RUN_ENDED, RUN_TIMEOUT,
  ACTION_STARTED, ACTION_COMPLETED,
  EVALUATOR_STARTED, EVALUATOR_COMPLETED,
  MODEL_USED,
  EMBEDDING_GENERATION_REQUESTED, EMBEDDING_GENERATION_COMPLETED, EMBEDDING_GENERATION_FAILED,
  CONTROL_MESSAGE,
}
```

Plugin registration:
```typescript
events: {
  [EventType.MESSAGE_RECEIVED]: [async (payload) => { /* payload.runtime, payload.message */ }],
}
```

All payloads extend `EventPayload { runtime, source, onComplete? }`. Uses `EventTarget` (Bun-native).

## Entity Component System (Replaces User System)

```typescript
interface Entity {
  id?: UUID; names?: string[];
  metadata?: Record<string, unknown>;
  agentId: UUID; components?: Component[];
}

interface Component {
  id?: UUID; entityId: UUID; agentId: UUID;
  roomId: UUID; worldId?: UUID;
  type: string; data: Record<string, unknown>;
}
```

Key functions: `createUniqueUuid(agentId, externalId)`, `findEntityByName()`, `getEntityDetails()`.

Plugins define component types:
```typescript
componentTypes: [{ name: 'wallet_info', schema: { address: { type: 'string' } } }]
```

## Task System

```typescript
interface TaskWorker {
  name: string;
  execute: (runtime, options, task) => Promise<void>;
  validate?: (runtime, message, state) => Promise<boolean>;
}
```

Tags: `queue` (picked up by TaskService), `repeat` (recurring), `immediate` (run ASAP).
TaskService polls every 1 second.

## ElizaOS Orchestrator (Multi-Agent)

```typescript
class ElizaOS extends EventTarget {
  addAgents(agents: ProjectAgent[], options?: {
    ephemeral?: boolean; autoStart?: boolean;
    returnRuntimes?: boolean; skipMigrations?: boolean;
  }): Promise<UUID[] | IAgentRuntime[]>;
  startAgents(ids?: UUID[]): Promise<void>;
  stopAgents(ids?: UUID[]): Promise<void>;
  handleMessage(agent, message, options?): Promise<HandleMessageResult>;
}
```

Two modes: **Sync** (blocks until response) and **Async** (callback-based).

## IAgentRuntime Key New Methods (vs v1.4.x)

```typescript
interface IAgentRuntime {
  // Model system
  useModel<T>(modelType, params, provider?): Promise<any>;
  registerModel(type, handler, provider, priority): void;
  generateText(params): Promise<string>;

  // Event system
  registerEvent(event, handler): void;
  emitEvent(event | event[], params): Promise<void>;

  // Task system
  registerTaskWorker(worker): void;
  getTaskWorker(name): TaskWorker | undefined;

  // Entity system (replaces user)
  ensureConnection(params): Promise<UUID>;
  getEntityById(id): Promise<Entity>;
  createEntity(entity): Promise<UUID>;

  // Service enhancements
  getServicesByType<T>(type): T[];
  getAllServices(): Map<string, Service[]>;
  hasService(type): boolean;
  getServiceLoadPromise(type): Promise<Service>;

  // Message routing
  registerSendHandler(platform, handler): void;
  sendMessageToTarget(target, content): Promise<void>;

  // Run tracking
  startRun(runId): void;
  endRun(runId): void;
  getCurrentRunId(): UUID | undefined;

  // Settings
  getSetting(key): string | boolean | number | null;
  setSetting(key, value): void;
}
```

## Settings Resolution Order (highest → lowest)

1. Request-context entity settings (per-entity, multi-tenant)
2. `character.secrets[key]`
3. `character.settings[key]`
4. `character.settings.secrets[key]`
5. `runtime.settings[key]` (from constructor)

Strings auto-decrypted (AES-256-CBC). `'true'`/`'false'` coerced to booleans.

## Database Schema

Drizzle ORM with PGLite (default), PostgreSQL, or Neon adapters.

**Embedding table:** 6 vector dimension columns (384, 512, 768, 1024, 1536, 3072) in one table.
**Row-Level Security:** Optional per-entity isolation via `ENABLE_DATA_ISOLATION=true`.

## Breaking Changes from v1.4.x → v2

| Area | v1.4.x | v2 |
|------|--------|-----|
| User system | `ensureUserExists()`, userId | Entity system: `ensureConnection()`, entityId |
| Service lifecycle | `new Service()` + `initialize()` | Static `Service.start(runtime)` returns instance |
| Service creation | Extend class only | + `createService()` / `defineService()` builders |
| Model usage | Direct API calls | `runtime.useModel()` with handler registry |
| Event system | None (ad-hoc) | Formal EventType enum + plugin event handlers |
| State | Flat `{ [key]: string }` | Structured `{ values, data, text }` |
| Memory | userId field | entityId field, MemoryType enum, MemoryScope |
| Action results | `void` / boolean | `ActionResult` with required `success` field |
| Action chaining | Not supported | `ActionContext` with `previousResults` |
| Plugin loading | Import objects | String names, auto-resolved with dependency sort |
| Database | SQLite adapter | Drizzle ORM: PGLite / PostgreSQL / Neon |
| Multi-agent | Not built-in | `ElizaOS` orchestrator class |
| Templates | String interpolation | Handlebars with `{{#if}}` conditionals |
| Testing | Vitest | `bun:test` |
| Worlds | Not present | World/Room/Entity hierarchy with roles |
| Task system | Not present | `TaskWorker` + persistent task queue |
| Run tracking | Not present | `createRunId()` / `startRun()` / `endRun()` |
| Settings encryption | Not present | AES-256-CBC for secrets |
| First-party plugins | In `packages/plugin-*` | Top-level `plugins/` directory (39 plugins on `develop`) |
| Runtime languages | TypeScript only | TypeScript + Python + Rust (shared protobuf wire format) |
