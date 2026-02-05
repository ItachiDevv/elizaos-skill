# ElizaOS v2.0.0 Architecture Reference

Branch: `v2.0.0` (194 commits ahead, 150 behind `develop`). Version: `2.0.0-alpha.2`. Node 23.3.0, Bun 1.3.5.

## Package Restructuring

```
packages/
  @schemas/        → Protobuf .proto schemas for cross-language type generation
  typescript/      → @elizaos/core (was packages/core/)
    src/
      advanced-capabilities/  → Extended agent capabilities
      advanced-memory/        → Advanced memory systems
      advanced-planning/      → Multi-step planning
      autonomy/               → Autonomous agent operations
      basic-capabilities/     → Core providers, actions, services
      bootstrap/              → Bootstrap plugin (moved FROM packages/plugin-bootstrap/)
      database/               → DB adapters (includes InMemoryAdapter)
      generated/              → Auto-generated code (action-docs, spec-helpers)
      schemas/                → Character schemas
      services/               → Message service, trajectory logger
      testing/                → Test framework
      types/                  → 26 type definition files
      utils/                  → buffer, environment, node, streaming
  python/          → Python runtime/SDK
  rust/            → Rust runtime/SDK
  interop/         → @elizaos/interop: cross-language plugin interop (TS/Python/Rust)
  elizaos/         → CLI binary (renamed from @elizaos/cli)
  computeruse/     → Computer use capabilities
  sweagent/        → Software engineering agent
  prompts/         → Standalone prompt templates
  docs/            → Documentation (Mintlify)

plugins/           → 45+ plugins at root level (moved from packages/plugin-*)
  plugin-agent-orchestrator, plugin-anthropic, plugin-auto-trader, plugin-blooio,
  plugin-bluesky, plugin-browser, plugin-code, plugin-computeruse, plugin-discord,
  plugin-goals, plugin-google-genai, plugin-groq, plugin-inmemorydb, plugin-instagram,
  plugin-knowledge, plugin-linear, plugin-local-ai, plugin-localdb, plugin-lp-manager,
  plugin-mcp, plugin-memory, plugin-minecraft, plugin-n8n, plugin-ollama, plugin-openai,
  plugin-openrouter, plugin-pdf, plugin-planning, plugin-polymarket, plugin-roblox,
  plugin-rss, plugin-s3-storage, plugin-scheduling, plugin-shell, plugin-simple-voice,
  plugin-solana, plugin-sql, plugin-tee, plugin-telegram, plugin-todo,
  plugin-trajectory-logger, plugin-twilio, plugin-vercel-ai-gateway, plugin-vision,
  plugin-whatsapp, plugin-xai
```

## Type System (26 files in packages/typescript/src/types/)

### Primitives (types/primitives.ts)
- **UUID**: String type
- **ChannelType**: SELF, DM, GROUP, VOICE_DM, VOICE_GROUP, FEED, THREAD, WORLD, FORUM, API (deprecated)
- **Content**: text, thoughts?, actions?, attachments?, channel?, metadata?, responseMessageId?
- **MentionContext**: isMention, isReply, isThread
- **Media**: id, url, title, source, description, contentType
- **ContentType**: IMAGE, VIDEO, AUDIO, DOCUMENT, LINK

### Agent (types/agent.ts)
```typescript
interface Character {
  id?: UUID; name: string; username?: string; system?: string;
  templates?: { [key: string]: TemplateType };
  bio: string | string[];
  messageExamples?: MessageExample[][];
  postExamples?: string[];
  topics?: string[]; adjectives?: string[];
  knowledge?: (string | { path: string; shared?: boolean } | DirectoryItem)[];
  plugins?: string[];
  settings?: CharacterSettings;
  secrets?: { [key: string]: string | boolean | number };
  style?: { all?: string[]; chat?: string[]; post?: string[] };
}

interface CharacterSettings {
  ENABLE_AUTONOMY?: boolean;
  DISABLE_BASIC_CAPABILITIES?: boolean;
  ENABLE_EXTENDED_CAPABILITIES?: boolean;
  ADVANCED_CAPABILITIES?: string[];
  secrets?: Record<string, string>;
  // ...extends proto settings
}
```
AgentStatus: ACTIVE, INACTIVE

### Memory (types/memory.ts)
```typescript
enum MemoryType { DOCUMENT, FRAGMENT, MESSAGE, DESCRIPTION, CUSTOM }
type MemoryScope = 'shared' | 'private' | 'room';

interface Memory {
  id?: UUID; createdAt?: string; embedding?: number[];
  metadata?: DocumentMetadata | FragmentMetadata | MessageMetadata | DescriptionMetadata | CustomMetadata;
  content: Content;
}
```
Type guards: isDocumentMetadata, isFragmentMetadata, isMessageMetadata, isDescriptionMetadata, isCustomMetadata

### Components (types/components.ts)
```typescript
interface Action {
  name: string; description: string; similes?: string[];
  examples?: ActionExample[][]; suppressInitialMessage?: boolean;
  validate(runtime: IAgentRuntime, message: Memory, state?: State): Promise<boolean>;
  handler(runtime: IAgentRuntime, message: Memory, state?: State,
    options?: HandlerOptions, callback?: HandlerCallback): Promise<ActionResult>;
}

interface Provider {
  name: string; description?: string; dynamic?: boolean;
  position?: number; private?: boolean;
  get(runtime: IAgentRuntime, message: Memory, state?: State): Promise<ProviderResult>;
}

interface Evaluator {
  name: string; description: string; similes?: string[];
  alwaysRun?: boolean; examples?: EvaluatorExample[];
  validate(runtime: IAgentRuntime, message: Memory, state?: State): Promise<boolean>;
  handler(runtime: IAgentRuntime, message: Memory, state?: State): Promise<any>;
}

interface ActionResult { success: boolean; text?: string; error?: string;
  values?: Record<string, any>; data?: Record<string, any>; }

interface HandlerOptions {
  actionContext?: { previousResults: ActionResult[]; currentStep: number; totalSteps: number; };
  actionPlan?: ActionPlan;
}
```

### Plugin (types/plugin.ts)
```typescript
interface Plugin {
  name: string; description?: string; priority?: number;
  dependencies?: string[]; testDependencies?: string[];
  init?(config: Record<string, string>, runtime: IAgentRuntime): Promise<void>;
  actions?: Action[]; providers?: Provider[]; evaluators?: Evaluator[];
  services?: ServiceClass[]; routes?: Route[]; events?: Record<string, Function[]>;
  tests?: TestSuite; config?: Record<string, any>; schema?: Record<string, any>;
  adapter?: IDatabaseAdapter; models?: Record<string, ModelHandler[]>;
}

interface Project { agents: ProjectAgent[] }
interface ProjectAgent { character: Character; init?(runtime: IAgentRuntime): Promise<void>;
  plugins?: (string | Plugin)[]; tests?: TestSuite; }
```

### Events (types/events.ts)
```
World: WORLD_JOINED, WORLD_CONNECTED, WORLD_LEFT
Entity: ENTITY_JOINED, ENTITY_LEFT, ENTITY_UPDATED
Room: ROOM_JOINED, ROOM_LEFT
Message: MESSAGE_RECEIVED, MESSAGE_SENT, MESSAGE_DELETED
Channel: CHANNEL_CLEARED
Voice: VOICE_MESSAGE_RECEIVED, VOICE_MESSAGE_SENT
Interaction: REACTION_RECEIVED, POST_GENERATED, INTERACTION_RECEIVED
Run: RUN_STARTED, RUN_ENDED, RUN_TIMEOUT
Action: ACTION_STARTED, ACTION_COMPLETED
Evaluator: EVALUATOR_STARTED, EVALUATOR_COMPLETED
Model: MODEL_USED
Embedding: EMBEDDING_GENERATION_REQUESTED/COMPLETED/FAILED
Control: CONTROL_MESSAGE
Form: FORM_FIELD_CONFIRMED, FORM_FIELD_CANCELLED
```
PlatformPrefix: DISCORD, TELEGRAM, X

### Models (types/model.ts)
```typescript
const ModelType = {
  TEXT_SMALL, TEXT_LARGE, TEXT_COMPLETION,
  TEXT_REASONING_SMALL, TEXT_REASONING_LARGE,  // NEW
  TEXT_EMBEDDING, TEXT_TOKENIZER_ENCODE, TEXT_TOKENIZER_DECODE,
  IMAGE, IMAGE_DESCRIPTION, TRANSCRIPTION, TEXT_TO_SPEECH,
  AUDIO, VIDEO, OBJECT_SMALL, OBJECT_LARGE,
  RESEARCH,  // NEW - deep research with web/file/code/MCP tools
} as const;

type LLMMode = 'DEFAULT' | 'SMALL' | 'LARGE';  // Runtime model override
```

MODEL_SETTINGS per-model-type: TEXT_SMALL_TEMPERATURE, TEXT_LARGE_MAX_TOKENS, etc.
VECTOR_DIMS: SMALL(384), MEDIUM(512), LARGE(768), XL(1024), XXL(1536), XXXL(3072)

### Services (types/service.ts)
```typescript
abstract class Service {
  runtime!: IAgentRuntime;
  static serviceType: string;
  capabilityDescription?: string;
  static start(runtime: IAgentRuntime): Promise<Service>;
  stop?(): Promise<void>;
}

const ServiceType = {
  transcription, video, browser, pdf, aws_s3, web_search, email, tee,
  task, wallet, lp_pool, token_data, message_service, message, post, unknown
} as const;
```

ServiceBuilder fluent API:
```typescript
createService<T>(serviceType).withDescription(desc).withStart(fn).withStop(fn).build()
defineService({ serviceType, description, start, stop })  // declarative
```

### State (types/state.ts)
```typescript
interface State {
  values: { agentName: string; actionNames: string; providers: string; [key: string]: unknown };
  data: StateData;
  text: string;
}

interface StateData {
  room?: Room; world?: World; entity?: Entity;
  providers?: Record<string, Record<string, unknown>>;
  actionPlan?: ActionPlan;
  actionResults?: ActionResult[];
  workingMemory?: Record<string, WorkingMemoryEntry>;
  [key: string]: unknown;
}

interface ActionPlan {
  thought: string; totalSteps: number; currentStep: number;
  steps: ActionPlanStep[];
}
interface ActionPlanStep {
  action: string; status: 'pending' | 'completed' | 'failed';
  error?: string; result?: ActionResult;
}
```

### Environment (types/environment.ts)
- **Entity**: id, names: string[], metadata, agentId, components?: Component[]
- **Component**: id, entityId, worldId, roomId, type, data, metadata, createdAt, updatedAt
- **World**: id, name, agentId, serverId, metadata (WorldMetadata)
- **Room**: id, name, source, type (ChannelType), channelId, messageServerId, worldId
- **Role**: OWNER, ADMIN, NONE
- **Relationship**: entityA, entityB, metadata, createdAt

### Messaging (types/messaging.ts)
- **SOCKET_MESSAGE_TYPE**: ROOM_JOINING(1), SEND_MESSAGE(2), MESSAGE(3), ACK(4), THINKING(5), CONTROL(6)
- **ControlMessage**: type 'control', payload { action: 'disable_input'|'enable_input' }
- **MessageResult**: messageId, userMessage, agentResponses
- **MESSAGE_STREAM_EVENT** for streaming chunks

### Streaming (types/streaming.ts)
- **IStreamExtractor**: { done: boolean; push(chunk: string): string; reset(): void; flush?(): string }
- Implementations: PassthroughExtractor, XmlTagExtractor (10-char safety margin), ResponseStreamExtractor
- **IStreamingRetryState**: getStreamedText, isComplete, reset

### Tasks (types/task.ts)
- **TaskWorker**: { name, execute(runtime, options, task), validate?(message, state) }
- **Task**: id, roomId, worldId, entityId, metadata (TaskMetadata), status, dueAt
- **TaskMetadata**: priority, updateInterval, scheduledAt, completedAt, options, values
- Tags: `queue` (one-time eligible), `repeat` (persists after execution), `immediate` (run ASAP)

### Payment (types/payment.ts) — NEW
x402 cryptocurrency payments: PaymentConfigDefinition, X402Config, X402Accepts, X402Response

### Database (types/database.ts)
- **IDatabaseAdapter<DB>**: Agent/Entity/Component/Memory/Embedding/Log/World/Room/Participant/Relationship/Cache/Task CRUD + migrations
- **RunStatus**: "started" | "completed" | "timeout" | "error"
- **AgentRunSummary**: Run tracking with timing metrics
- Log types: ActionLogBody, ModelLogBody, EvaluatorLogBody, EmbeddingLogBody

### Runtime (types/runtime.ts)
IAgentRuntime extends IDatabaseAdapter. Key methods:
- processActions, composeState, evaluate, ensureConnection
- getService<T>, getServicesByType<T>, getAllServices, hasService, getServiceLoadPromise
- useModel<T>(type, params, provider?), registerModel, getModel
- registerSendHandler(source, handler), sendMessageToTarget
- startRun, endRun, getCurrentRunId
- registerEvent<T>, emitEvent<T>
- queueEmbeddingGeneration(memory, priority?)
- getSetting, setSetting, isActionPlanningEnabled, getLLMMode
- registerTaskWorker, getTaskWorker

## AgentRuntime Class (runtime.ts — 4076 lines)

Key properties:
```typescript
class AgentRuntime implements IAgentRuntime {
  agentId: UUID; character: Character; adapter: IDatabaseAdapter;
  actions: Action[]; evaluators: Evaluator[]; providers: Provider[]; plugins: Plugin[];
  events: RuntimeEventStorage;
  services: Map<ServiceTypeName, Service[]>;
  models: Map<string, ModelHandler[]>;
  routes: Route[];
  messageService: IMessageService | null;
  enableAutonomy: boolean;
  maxWorkingMemoryEntries: number; // default: 50
  private servicePromises: Map<string, Promise<Service>>;
  private currentRunId?: UUID;
}
```

Constructor options: conversationLength, agentId, character, plugins, adapter, settings, logLevel, disableBasicCapabilities, enableExtendedCapabilities, actionPlanning, llmMode, checkShouldRespond, enableAutonomy.

## Bootstrap Plugin — Capability Tiers

Created via `createBootstrapPlugin(config?: CapabilityConfig)`.

**Basic (default):**
- Providers: actions, actionState, attachments, capabilities, character, contextBench, entities, evaluators, providers, recentMessages, time, world
- Actions: reply, ignore, none
- Services: TaskService, EmbeddingGenerationService, TrajectoryLoggerService

**Extended (ENABLE_EXTENDED_CAPABILITIES):**
- +Providers: choice, contacts, facts, followUps, knowledge, relationships, role, settings
- +Actions: addContact, choice, followRoom, generateImage, muteRoom, removeContact, scheduleFollowUp, searchContacts, sendMessage, unfollowRoom, unmuteRoom, updateContact, updateEntity, updateRole, updateSettings
- +Evaluators: reflection, relationshipExtraction
- +Services: RolodexService, FollowUpService

**Autonomy (ENABLE_AUTONOMY):**
- +Providers: adminChat, autonomyStatus
- +Actions: sendToAdmin
- +Services: AutonomyService
- +Routes: autonomyRoutes

Event handlers: REACTION_RECEIVED, POST_GENERATED, MESSAGE_SENT, WORLD_JOINED, WORLD_CONNECTED, ENTITY_JOINED, ENTITY_LEFT, ACTION_STARTED, ACTION_COMPLETED, EVALUATOR_STARTED, EVALUATOR_COMPLETED, RUN_STARTED, RUN_ENDED, RUN_TIMEOUT, CONTROL_MESSAGE

## Message Service (DefaultMessageService)

Two processing modes:
- **Single-Shot** (`runSingleShotCore`): One LLM call → thought + actions + response. Retry for missing fields. Auto parameter repair.
- **Multi-Step** (`runMultiStepCore`): Iterative workflow with accumulated context, provider timeout protection, summary generation.

Response decision: DM/voice/API auto-respond. Platform mentions bypass eval. Others defer to LLM. Configurable via SHOULD_RESPOND_BYPASS_TYPES/SOURCES.

Options: maxRetries, timeoutDuration, useMultiStep, maxMultiStepIterations, shouldRespondModel, onStreamChunk.

## Prompt Templates (prompts.ts — Handlebars syntax)

Decision: shouldRespondTemplate, messageHandlerTemplate
Content: postCreationTemplate, replyTemplate, imageDescriptionTemplate
Multi-step: multiStepDecisionTemplate, multiStepSummaryTemplate
Contact: scheduleFollowUpTemplate, addContactTemplate, searchContactsTemplate, updateContactTemplate, removeContactTemplate
Room: shouldFollowRoomTemplate, shouldMuteRoomTemplate, shouldUnfollowTemplate, shouldUnmuteTemplate
Memory: initialSummarizationTemplate, updateSummarizationTemplate, longTermExtractionTemplate, reflectionEvaluatorTemplate
Entity: entityResolutionTemplate, componentTemplate
Settings: settingsSuccessTemplate, settingsFailureTemplate, settingsErrorTemplate, settingsCompletionTemplate
Autonomy: autonomyContinuousFirstTemplate, autonomyContinuousContinueTemplate, autonomyTaskFirstTemplate, autonomyTaskContinueTemplate

Key pattern: Action ordering (REPLY first), XML-only responses, provider selection, parameter extraction in `<params>` blocks. UPPERCASE aliases for backward compatibility.

## Settings & Security

- AES-256-GCM encryption (v2) with migration from v1 AES-256-CBC
- SECRET_SALT management with 5-minute TTL caching
- World settings auto-encrypted/decrypted in metadata
- `initializeOnboarding()` for new server setup

## Search (search.ts)

Full BM25 implementation: Porter2 stemming, stop words, Unicode normalization, emoji removal, phrase search with sliding-window, configurable k1/b/field boosts.

## InMemoryAdapter (database/inMemoryAdapter.ts)

For benchmarks, tests, serverless/ephemeral runs. Maps for all entities, bidirectional participant-room lookups, cascading deletions.

## Breaking Changes from v1 → v2

1. Package restructuring (packages/core → packages/typescript, etc.)
2. CLI renamed: @elizaos/cli → elizaos package
3. Bootstrap integrated into core (not separate plugin-bootstrap)
4. Memory types: 7 → 5 (DOCUMENT, FRAGMENT, MESSAGE, DESCRIPTION, CUSTOM)
5. Memory scope added (shared, private, room)
6. Protobuf base types (all types extend proto-generated, omit $typeName/$unknown)
7. Entity component system for metadata
8. New events (embedding, form, channel), some renamed (RUN_ENDED, RUN_TIMEOUT)
9. Server/Client packages removed as top-level
10. Plugins moved to root-level plugins/ directory
11. ServiceBuilder fluent API alongside class extension pattern
12. Multi-language SDKs (Python, Rust) via interop package
