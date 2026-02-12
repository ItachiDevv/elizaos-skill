# ElizaOS Core Documentation - Complete Extraction

---

## 1. CHARACTER INTERFACE (https://docs.elizaos.ai/agents/character-interface)

### Core Concept

A Character is a configuration blueprint that defines an agent's personality, capabilities, and settings. At runtime, this Character transforms into an Agent -- a living instance with status tracking and lifecycle management.

### Minimum Required Setup

```typescript
export const character: Character = {
  name: "Chef Mario",
  bio: "A passionate Italian chef who loves sharing recipes and cooking tips.",
  plugins: ["@elizaos/plugin-openai"],
};
```

### Character Interface (Full TypeScript Definition)

```typescript
interface Character {
  name: string;                    // Required - Display identifier
  bio: string | string[];          // Required - Personality/background
  id?: UUID;                       // Auto-generated if omitted
  username?: string;               // Social platform handle
  system?: string;                 // System prompt override
  templates?: object;              // Custom prompt templates
  adjectives?: string[];           // Trait descriptors
  topics?: string[];               // Knowledge domains
  knowledge?: array;               // Facts/files/directories
  messageExamples?: array[][];     // Conversation samples
  postExamples?: string[];         // Social media samples
  style?: object;                  // Context-specific writing styles
  plugins?: string[];              // Enabled packages
  settings?: object;               // Configuration values
  secrets?: object;                // Sensitive credentials
}
```

### Agent Interface (extends Character)

```typescript
interface Agent extends Character {
  enabled?: boolean;
  status?: 'active' | 'inactive';
  createdAt: number;
  updatedAt: number;
}
```

### Templates Structure

```typescript
templates?: {
  messageTemplate?: string | ((params: any) => string);
  thoughtTemplate?: string | ((params: any) => string);
  actionTemplate?: string | ((params: any) => string);
  [key: string]: string | ((params: any) => string);
}
```

### Message Examples Structure

```typescript
messageExamples: [
  [
    {
      name: string;           // "{{user}}" for user, or agent name
      content: { text: string };
    }
  ]
]
```

### Knowledge Array Item Types

```typescript
knowledge: [
  string,                          // Simple string facts
  {
    path: string;                  // File reference
    shared: boolean;
  },
  {
    directory: string;             // Directory reference
    shared: boolean;
  }
]
```

### Style Object

```typescript
style: {
  all?: string[];    // Universal style rules
  chat?: string[];   // Conversational style rules
  post?: string[];   // Social media style rules
}
```

### Settings Object

```typescript
settings: {
  model?: string;
  temperature?: number;
  maxTokens?: number;
  responseTimeout?: number;
  maxMemorySize?: number;
  modelProvider?: string;          // 'openai', 'anthropic', 'llama', etc.
  [key: string]: any;
}
```

### Secrets Object

```typescript
secrets: {
  [key: string]: string | undefined;   // API keys, tokens from env vars
}
```

### Bio Format Options

Single string:
```typescript
bio: "An expert in web development"
```

Array format (each element is randomly sampled during context composition):
```typescript
bio: [
  "Expert in web development",
  "Specializes in TypeScript"
]
```

### Conditional Plugin Loading

```typescript
plugins: [
  ...(process.env.OPENAI_API_KEY ? ["@elizaos/plugin-openai"] : []),
]
```

### Validation

```typescript
validateCharacter(character)  // Returns valid status; use before production deployment
```

### Best Practices

1. Maintain consistency between bio, adjectives, and style traits
2. Load plugins conditionally based on available API keys
3. Never hardcode sensitive data; use environment variables
4. Validate characters before deployment using validateCharacter()
5. Order plugins by dependency hierarchy
6. Provide diverse message examples covering multiple interaction patterns
7. Document any custom configuration settings
8. Test conversation flows with provided examples

---

## 2. MEMORY AND STATE (https://docs.elizaos.ai/agents/memory-and-state)

### Core Architecture

The system centers on AgentRuntime handling unified memory creation, storage, retrieval, and search. The flow:
1. Process user messages
2. Generate embeddings
3. Store in database
4. Retrieve via time-based, vector semantic, or keyword search
5. Compose context for LLM consumption

### Memory Interface

```typescript
interface Memory {
  id?: UUID;
  entityId: UUID;          // Who created this memory (user or agent)
  agentId?: UUID;          // Which agent owns this memory
  roomId: UUID;            // Conversation/room context
  worldId?: UUID;          // World-level grouping
  content: Content;        // The actual content
  embedding?: number[];    // Vector embedding for semantic search
  createdAt?: number;      // Timestamp
  unique?: boolean;        // Deduplication flag
  similarity?: number;     // Search result similarity score
  metadata?: MemoryMetadata;
}

interface Content {
  text?: string;
  actions?: string[];
  inReplyTo?: UUID;
  metadata?: any;
}
```

### State Interface

```typescript
interface State {
  [key: string]: unknown;
  values: {
    [key: string]: unknown;
  };
  data: StateData;
  text: string;
}

interface StateData {
  room?: Room;
  world?: World;
  entity?: Entity;
  providers?: Record<string, Record<string, unknown>>;
  actionPlan?: ActionPlan;
  actionResults?: ActionResult[];
  [key: string]: unknown;
}
```

### Memory Lifecycle Operations

**Creation:**
```typescript
createMemory(memory: Memory, tableName: string, unique?: boolean): Promise<UUID>
```

**Retrieval (various patterns):**
```typescript
// Recent messages
getMemories({ roomId, count: 10, unique: true })

// User-specific memories
getMemories({ entityId, count: 20 })

// Time-bounded retrieval
getMemories({ roomId, start, end })

// Full parameter signature
getMemories({ roomId, count, entityId, start, end, offset, orderBy, direction }): Promise<Memory[]>
```

**State Composition:**
```typescript
composeState(
  message: Memory,
  includeList?: string[],     // Provider names to include
  onlyInclude?: boolean,      // If true, ONLY include listed providers
  skipCache?: boolean         // Force fresh state composition
): Promise<State>
```

### Context Management

- Default `conversationLength = 32` messages
- Dynamic adjustment based on token limits
- Context strategies:
  - Recency-based: Most recent messages prioritized
  - Importance-based: Filtered by significance scores
  - Hybrid: Combines both approaches with deduplication

### Search Operations

**Semantic Search:**
```typescript
searchMemories({
  embedding,
  match_threshold: 0.75,
  count: 10,
  roomId
}): Promise<Memory[]>
```

**Hybrid Search:**
Combines semantic vector search with keyword extraction; results ranked together.

**Vector Search:**
Uses cosine similarity with optional ANN (Approximate Nearest Neighbor) indexing for scalability.

### Embedding Management

```typescript
class EmbeddingManager {
  private model: EmbeddingModel;
  private cache = new Map<string, number[]>();

  async generateEmbedding(text: string): Promise<number[]>
  async generateBatch(texts: string[]): Promise<number[][]>
}
```

- Caching prevents duplicate computation
- Batch processing handles multiple texts efficiently

### Memory Types

| Type | Description | Details |
|------|-------------|---------|
| Short-term | Working buffer | Max 50, FIFO eviction |
| Long-term | Persistent | Marked with importance, access counts, consolidation timestamps |
| Knowledge | Static + dynamic | Character-defined facts plus dynamically learned items |

### Multi-level Caching

| Level | Type | Details |
|-------|------|---------|
| L1 | Hot in-memory cache | Immediate access |
| L2 | LRU cache | 5-minute TTL, max 1000 entries |
| Database | Persistent storage | Full durability |

### Database Optimization

- BTree indexes for roomId/entityId/createdAt
- IVF index for embeddings
- Batch transactions for bulk operations

### Pruning Strategies

- Time-based: Remove items older than threshold (preserve long-term flagged)
- Importance-based: Delete lowest-scoring memories maintaining max count

### Memory Networks (Advanced)

Graph structures linking memories via connections:
- cause
- effect
- relation
- reference

### Temporal Patterns

- Window-based retrieval
- Exponential decay modeling (7-day half-life)

### Multi-agent Memory

- Shared spaces with granular permissions
- Per-agent read/write/delete access control

### Troubleshooting

- Search debugging: Test multiple thresholds (0.9 down to 0.5), validate embedding generation, verify memory existence
- Memory monitoring: Track total count, average size, growth rates; alert on 10%+ growth

---

## 3. PERSONALITY AND BEHAVIOR (https://docs.elizaos.ai/agents/personality-and-behavior)

### Core Principles

1. **Consistency Over Complexity**: A simple, consistent personality is better than a complex, contradictory one
2. **Purpose-Driven Design**: Every personality trait should support the agent's primary function
3. **Cultural Awareness**: Consider cultural contexts and sensitivities
4. **Evolutionary Potential**: Design personalities that can grow and adapt

### Bio Formats

**Single String:**
```typescript
bio: "A helpful AI assistant specializing in customer support"
```

**Array Format:**
```typescript
bio: [
  "Former software engineer turned AI educator",
  "Passionate about making technology accessible to everyone",
  "Specializes in web development and cloud architecture",
  "Believes in learning through practical examples",
  "Fluent in multiple programming languages"
]
```

### Style Configuration (Three Contexts)

```typescript
style: {
  all: [
    "Be clear and concise",
    "Use active voice",
    "Avoid jargon unless necessary",
    "Include examples when explaining concepts",
    "Admit uncertainty when appropriate"
  ],
  chat: [
    "Be conversational but professional",
    "Use markdown for code formatting",
    "Break long explanations into digestible chunks",
    "Ask clarifying questions",
    "Use appropriate emoji to add warmth (sparingly)"
  ],
  post: [
    "Hook readers in the first line",
    "Use line breaks for readability",
    "Include relevant hashtags (3-5 max)",
    "End with a call to action or question",
    "Keep under platform limits"
  ]
}
```

### Adjectives (Well-Balanced Sets)

```typescript
// Helper personality
adjectives: ["helpful", "patient", "knowledgeable", "approachable", "reliable"]

// Creative personality
adjectives: ["creative", "innovative", "bold", "inspiring", "unconventional"]

// Analyst personality
adjectives: ["analytical", "precise", "methodical", "thorough", "objective"]
```

**Contradictory combinations to avoid:**
```typescript
// BAD: ["aggressive", "gentle", "pushy", "caring"]
// GOOD: ["assertive", "supportive", "confident", "encouraging"]
```

### Topics (Domain Expertise)

```typescript
topics: [
  // Core expertise
  "JavaScript", "TypeScript", "React", "Node.js",
  // Secondary knowledge
  "web performance", "SEO basics", "UI/UX principles",
  // Peripheral awareness
  "tech industry trends", "programming history"
]
```

### Behavioral Consistency Matrix

| Trait | Bio Expression | Message Style | Post Style |
|-------|---------------|---------------|------------|
| Helpful | "Dedicated to user success" | Asks clarifying questions | Shares useful tips |
| Expert | "15 years experience" | Provides detailed explanations | Shares industry insights |
| Friendly | "Approachable mentor" | Uses warm greetings | Includes community engagement |
| Analytical | "Data-driven approach" | Breaks down problems | Cites statistics and research |

### Message Examples

**Greeting Style:**
```typescript
messageExamples: [
  [
    { name: "{{user}}", content: { text: "Hi" } },
    { name: "Assistant", content: { text: "Hello! Ready to build something amazing today?" } }
  ]
]
```

**Problem-Solving Approach:**
```typescript
[
  { name: "{{user}}", content: { text: "My code isn't working" }},
  { name: "Assistant", content: {
    text: "I'm here to help! Let's debug this together. Can you tell me:\n1. What you expected\n2. What actually happened\n3. Any error messages?"
  }}
]
```

**Knowledge Boundaries:**
```typescript
[
  { name: "{{user}}", content: { text: "Can you help with quantum computing?" }},
  { name: "Assistant", content: {
    text: "While I have basic knowledge of quantum concepts, it's not my specialty. I'm most helpful with web development, cloud architecture, and general programming. Would you prefer one of those areas or shall I share what I know about quantum computing?"
  }}
]
```

### Dynamic Response Templates

```typescript
templates: {
  greeting: ({ timeOfDay }) => {
    const greetings = {
      morning: "Good morning! Ready to code?",
      afternoon: "Good afternoon! How's your project going?",
      evening: "Good evening! Still coding? I'm here to help!",
      night: "Hey night owl! What are we building?"
    };
    return greetings[timeOfDay];
  },
  errorHelp: ({ errorType, context }) => {
    return `I see you're encountering a ${errorType} error. This often happens when ${context}. Let's debug this step by step.`;
  },
  success: ({ achievement }) => {
    const celebrations = [
      `Brilliant! You ${achievement}!`,
      `Excellent work! ${achievement} is no small feat!`,
      `You did it! ${achievement} - that's fantastic!`
    ];
    return celebrations[Math.floor(Math.random() * celebrations.length)];
  }
}
```

### Personality Archetypes

**The Helper:**
```typescript
export const helperCharacter: Character = {
  name: "SupportBot",
  bio: [
    "Your dedicated support companion",
    "Available 24/7 to solve problems",
    "Patient, thorough, and always friendly"
  ],
  adjectives: ["helpful", "patient", "thorough", "friendly", "reliable"],
  topics: ["troubleshooting", "guidance", "support", "solutions"],
  style: {
    all: ["Focus on solving the user's immediate problem", "Be patient with frustrated users"],
    chat: ["Always acknowledge the user's concern first", "Provide step-by-step solutions"],
    post: ["Share helpful tips and common solutions", "Create troubleshooting guides"]
  },
  messageExamples: [[
    { name: "{{user}}", content: { text: "Nothing is working!" }},
    { name: "SupportBot", content: {
      text: "I understand how frustrating that must be. Let's get this sorted out together. Can you tell me what specific issue you're facing first?"
    }}
  ]]
}
```

**The Expert:**
```typescript
export const expertCharacter: Character = {
  name: "TechExpert",
  bio: [
    "Senior architect with 20 years experience",
    "Published author and conference speaker",
    "Specializes in scalable system design"
  ],
  adjectives: ["knowledgeable", "analytical", "precise", "authoritative", "insightful"],
  topics: ["architecture", "performance", "scalability", "best practices", "design patterns"],
  style: {
    all: ["Provide technically accurate information", "Reference authoritative sources"],
    chat: ["Lead with expertise", "Offer multiple solution approaches"],
    post: ["Share advanced techniques", "Discuss industry trends"]
  },
  messageExamples: [[
    { name: "{{user}}", content: { text: "Should I use microservices?" }},
    { name: "TechExpert", content: {
      text: "Microservices aren't a silver bullet. They solve specific problems: independent scaling, team autonomy, and technology diversity. But they introduce complexity in service discovery, data consistency, and operational overhead. For most applications under 100k daily users, a well-structured monolith is more maintainable. What's your current scale and pain points?"
    }}
  ]]
}
```

**The Companion:**
```typescript
export const companionCharacter: Character = {
  name: "DevBuddy",
  bio: [
    "Your coding companion and cheerleader",
    "Here for the late-night debugging sessions",
    "Celebrates your wins, supports through challenges"
  ],
  adjectives: ["empathetic", "encouraging", "warm", "supportive", "understanding"],
  topics: ["motivation", "learning", "growth", "wellbeing", "community"],
  style: {
    all: ["Show genuine care for user wellbeing", "Celebrate small victories"],
    chat: ["Check in on user's emotional state", "Provide encouragement"],
    post: ["Share motivational content", "Build community connections"]
  },
  messageExamples: [[
    { name: "{{user}}", content: { text: "I've been stuck on this bug for hours" }},
    { name: "DevBuddy", content: {
      text: "Those marathon debugging sessions are tough! You're showing real persistence. Sometimes a quick break helps our brains process in the background. Want to talk through what you've tried so far? Fresh perspective might help!"
    }}
  ]]
}
```

**The Analyst:**
```typescript
export const analystCharacter: Character = {
  name: "DataAnalyst",
  bio: [
    "Data scientist turned AI analyst",
    "Decisions backed by metrics and research",
    "Objective, thorough, evidence-based"
  ],
  adjectives: ["analytical", "objective", "methodical", "logical", "data-driven"],
  topics: ["metrics", "analysis", "optimization", "research", "statistics"],
  style: {
    all: ["Support claims with data", "Present multiple perspectives objectively"],
    chat: ["Ask for metrics and constraints", "Provide quantitative comparisons"],
    post: ["Share data visualizations", "Discuss research findings"]
  },
  messageExamples: [[
    { name: "{{user}}", content: { text: "Which framework is faster?" }},
    { name: "DataAnalyst", content: {
      text: "Based on the latest benchmarks: React re-renders at ~16ms for a typical component tree, while Vue 3's reactive system achieves ~12ms. However, real-world performance depends on: bundle size (React: 42kb, Vue: 34kb), your specific use case, and optimization techniques. Would you like to see performance metrics for your particular scenario?"
    }}
  ]]
}
```

### Knowledge Integration with plugin-knowledge

```typescript
export const shakespeareScholar: Character = {
  name: "Shakespeare Scholar",
  plugins: [
    '@elizaos/plugin-openai',
    '@elizaos/plugin-knowledge',
  ],
  knowledge: [
    "I am an expert in William Shakespeare's complete works",
    "I specialize in Elizabethan theater and Renaissance literature",
    "My expertise covers all 39 plays, 154 sonnets, and narrative poems"
  ]
}
```

**Document Folder Structure for knowledge plugin:**
```
your-project/
├── docs/
│   ├── shakespeare/
│   │   ├── complete-works.pdf
│   │   ├── sonnets.txt
│   │   └── plays/
│   │       ├── hamlet.md
│   │       ├── macbeth.md
│   │       └── romeo-juliet.md
│   ├── criticism/
│   │   ├── bloom-analysis.pdf
│   │   └── bradley-tragic-hero.docx
│   └── history/
│       ├── elizabethan-context.md
│       └── globe-theatre.json
├── .env
├── src/
│   └── character.ts
```

**Required Environment Variables for Knowledge:**
```env
OPENAI_API_KEY=sk-...
LOAD_DOCS_ON_STARTUP=true
CTX_KNOWLEDGE_ENABLED=true
KNOWLEDGE_PATH=/path/to/custom/docs
MAX_INPUT_TOKENS=4000
MAX_OUTPUT_TOKENS=2000
```

**Important Note:** The `knowledge` array is only for small snippets. For actual documents, use the `docs` folder with `LOAD_DOCS_ON_STARTUP=true`.

### Multi-Persona Agents

```typescript
templates: {
  personaSwitch: ({ mode }) => {
    const personas = {
      teacher: "Let me explain this step-by-step...",
      expert: "From an architectural perspective...",
      friend: "Hey! Let's figure this out together...",
      coach: "You've got this! Here's how to approach it..."
    };
    return personas[mode];
  }
}
```

### Personality Validation Checklist

- [ ] Bio aligns with adjectives
- [ ] Message examples demonstrate stated traits
- [ ] Style rules don't contradict personality
- [ ] Topics match claimed expertise
- [ ] Post examples fit the character voice
- [ ] Knowledge supports the backstory
- [ ] No conflicting behavioral patterns

### Testing Personality Consistency

```typescript
describe('Personality Consistency', () => {
  it('should maintain consistent tone across contexts', () => {
    const responses = generateResponses(character, ['chat', 'post']);
    responses.forEach(response => {
      expect(response).toMatchPersonalityTraits(character.adjectives);
    });
  });

  it('should demonstrate claimed expertise', () => {
    const technicalResponse = generateResponse(character, "Explain async/await");
    expect(technicalResponse).toShowExpertise(character.topics);
  });

  it('should handle edge cases consistently', () => {
    const edgeCases = [
      "I don't understand",
      "You're wrong",
      "Can you help with [unrelated topic]?"
    ];
    edgeCases.forEach(input => {
      const response = generateResponse(character, input);
      expect(response).toMaintainPersonality(character);
    });
  });
});
```

---

## 4. RUNTIME AND LIFECYCLE (https://docs.elizaos.ai/agents/runtime-and-lifecycle)

### Agent Lifecycle Phases

1. **Configuration Phase**: Character definition and validation
2. **Initialization Phase**: Runtime creation, database connection, plugin loading, service startup
3. **Runtime Phase**: Message processing loop, action execution, state management
4. **Shutdown Phase**: Graceful termination and resource cleanup

### Character Loading Sources

- TypeScript files
- JSON files
- Environment variables
- Remote URLs

### Validation Requirements

- Non-empty `name` (required)
- Non-empty `bio` (required)
- Defaults applied for optional fields: topics, adjectives, messageExamples, style

### AgentRuntime Class

```typescript
export class AgentRuntime implements IAgentRuntime {
  readonly agentId: UUID;
  readonly character: Character;
  public adapter!: IDatabaseAdapter;
  readonly actions: Action[] = [];
  readonly evaluators: Evaluator[] = [];
  readonly providers: Provider[] = [];
  readonly plugins: Plugin[] = [];
  services = new Map<ServiceTypeName, Service[]>();
  models = new Map<string, ModelHandler[]>();

  constructor(opts: {
    character: Character;
    adapter?: IDatabaseAdapter;
    plugins?: Plugin[];
    settings?: RuntimeSettings;
  });

  async initialize(): Promise<void>;
}
```

### Initialization Sequence

```typescript
async initialize(): Promise<void> {
  // 1. Database Connection
  await this.adapter.init();

  // 2. Plugin Resolution (topological sort by dependency graph)
  const pluginsToLoad = await this.resolvePluginDependencies(this.characterPlugins);
  const sortedPlugins = this.topologicalSort(pluginsToLoad);

  // 3. Sequential Plugin Loading
  for (const plugin of sortedPlugins) {
    await this.registerPlugin(plugin);
  }

  // 4. Service Startup
  await this.startServices();

  // 5. Provider Initialization
  await this.initializeProviders();

  // 6. Status Update
  await this.updateAgentStatus(AgentStatus.ACTIVE);

  this.isInitialized = true;
}
```

### Plugin Interface

```typescript
interface Plugin {
  name: string;
  dependencies?: string[];
  priority?: number;
  config?: any;

  // Lifecycle hooks
  init?: (config: any, runtime: IAgentRuntime) => Promise<void>;
  start?: (runtime: IAgentRuntime) => Promise<void>;
  stop?: (runtime: IAgentRuntime) => Promise<void>;

  // Message processing hooks
  beforeMessage?: (message: Memory, runtime: IAgentRuntime) => Promise<Memory>;
  afterMessage?: (message: Memory, response: Memory, runtime: IAgentRuntime) => Promise<void>;

  // Action hooks
  beforeAction?: (action: Action, message: Memory, runtime: IAgentRuntime) => Promise<boolean>;
  afterAction?: (action: Action, result: any, runtime: IAgentRuntime) => Promise<void>;

  // Component registrations
  services?: typeof Service[];
  actions?: Action[];
  providers?: Provider[];
  evaluators?: Evaluator[];
  models?: Record<string, any>;
}
```

### Service Abstract Class

```typescript
abstract class Service {
  static serviceType: ServiceTypeName;
  status: ServiceStatus = ServiceStatus.STOPPED;
  runtime: IAgentRuntime;

  constructor(runtime: IAgentRuntime);
  abstract start(): Promise<void>;
  abstract stop(): Promise<void>;
}
```

Services follow dependency-based ordering during startup and reverse ordering during shutdown. Services can error without failing initialization if non-critical.

### State Composition (Provider Execution)

```typescript
async composeState(
  runtime: IAgentRuntime,
  message: Memory
): Promise<State> {
  const state: State = {
    messages: await runtime.getMemories({ ... }),
    facts: [],
    providers: {},
    context: ''
  };

  // Providers run in PARALLEL
  const providerPromises = runtime.providers.map(async provider => {
    const result = await provider.get(runtime, message, state);
    return { name: provider.name, result };
  });

  const providerResults = await Promise.all(providerPromises);
  // Merge provider outputs into unified state
}
```

### Action Selection and Execution

```typescript
async selectAction(
  runtime: IAgentRuntime,
  message: Memory,
  state: State
): Promise<Action | null> {
  // Validate available actions against incoming message
  const validActions = await Promise.all(
    availableActions.map(action => action.validate?.(runtime, message, state))
  );

  // Use LLM to select best candidate
  return await this.selectWithLLM(runtime, candidates, message, state);
}

async executeAction(
  runtime: IAgentRuntime,
  action: Action,
  message: Memory,
  state: State
): Promise<ActionResult> {
  // beforeAction hook
  const result = await action.handler(runtime, message, state, {}, callback);
  // afterAction hook
  return { success: true, data: result };
}
```

### Evaluator Post-Processing

```typescript
async runEvaluators(
  runtime: IAgentRuntime,
  message: Memory,
  response: Memory,
  state: State
): Promise<EvaluationResults> {
  const evaluatorsToRun = runtime.evaluators.filter(evaluator =>
    evaluator.alwaysRun || response
  );

  // Evaluators run in PARALLEL
  const evaluations = await Promise.all(
    evaluatorsToRun.map(evaluator =>
      evaluator.handler(runtime, message, state, {}, () => {}, [response])
    )
  );
}
```

### Component Registration Methods

```typescript
registerAction(action: Action): void;
registerProvider(provider: Provider): void;
registerEvaluator(evaluator: Evaluator): void;

async registerService(ServiceClass: typeof Service): Promise<Service> {
  const serviceName = ServiceClass.serviceType;

  if (this.services.has(serviceName)) {
    return this.services.get(serviceName)[0];
  }

  const service = new ServiceClass(this);
  await service.start();
  this.services.set(serviceName, [service]);
  return service;
}

getService<T extends Service>(name: ServiceTypeName): T | null;
```

### Error Recovery Pattern

```typescript
class ResilientRuntime extends AgentRuntime {
  async processMessageWithRecovery(message: Memory): Promise<void> {
    const maxRetries = 3;

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        const breaker = this.circuitBreakers.get('message_processing');
        if (breaker?.isOpen()) throw new Error('Circuit breaker open');

        await this.processMessage(message);
        this.errorCount.set('message_processing', 0);
        return;
      } catch (error) {
        const count = (this.errorCount.get('message_processing') || 0) + 1;
        this.errorCount.set('message_processing', count);

        if (count > 10) {
          this.circuitBreakers.get('message_processing')?.trip();
        }

        if (attempt < maxRetries) {
          await this.sleep(Math.pow(2, attempt) * 1000); // Exponential backoff
        }
      }
    }

    await this.handleCriticalError(lastError, message);
  }
}
```

### Multi-Agent Systems

- `MultiAgentCoordinator` handles agent-to-agent communication
- Message routing and queuing between agents
- Parent-child hierarchies with delegation, inherited settings, permission management
- Agents share database adapters but maintain independent message processing

### Deployment Patterns

**Single Agent:**
- HTTP server wrapping a runtime instance with health check endpoints

**Agent Swarm:**
- Multiple agents with shared database
- Message bus for inter-agent communication

**Edge Deployment:**
- SQLite adapter
- Reduced memory buffers
- Shorter conversation context
- Offline mode with cloud sync

### Production Strategies

**Lazy Initialization:** Load core plugins only, fetch additional on-demand
**Eager Initialization:** Pre-load all plugins, warm caches, compile templates upfront

**Monitoring Metrics:**
- Message count
- Response time
- Error rates
- Memory usage
- Per-plugin performance

**Scaling:** Round-robin or load-based routing; horizontal scaling by adding/removing instances

### Best Practices

1. Create runtime once, reuse for all operations
2. Always call stop() for graceful shutdown
3. Implement health checks and metrics collection
4. Use dependency injection for component access
5. Prevent cascading failures with circuit breakers
6. Cache expensive operations with memory management
7. Version deployments for tracking
8. Test in production-like environments
9. Plan for failure with fallbacks
10. Avoid logging sensitive data

---

## 5. PROJECTS OVERVIEW (https://docs.elizaos.ai/projects/overview)

### Core Definition

Projects are the primary deployable unit in elizaOS. Each project contains one or more agents, and plugins can be managed either at the agent level or shared across the project. A project serves as both your development workspace and deployment container.

### Key Relationships

- A **Project** has one or many **Agents**
- **Agents** import/use **Plugins**
- **Plugins** are reusable across projects and agents

### Development Workflow

1. `elizaos create --type project` -- Initialize workspace
2. Define agent personalities and select plugins
3. Build custom plugins and actions within project context
4. `elizaos start` -- Run/test locally (without full monorepo)
5. Deploy complete project to production

### Project Interface and Types

```typescript
import { Project, ProjectAgent, IAgentRuntime } from '@elizaos/core';
```

### Single-Agent Project (src/index.ts)

```typescript
import { Project, ProjectAgent, IAgentRuntime } from '@elizaos/core';
import { character } from './character';

export const projectAgent: ProjectAgent = {
  character,
  init: async (runtime: IAgentRuntime) => {
    console.log(`Initializing ${character.name}`);
    // Optional: Setup knowledge base, connections, etc.
  },
  // plugins: [customPlugin], // Project-specific plugins
};

const project: Project = {
  agents: [projectAgent],
};

export default project;
```

### Multi-Agent Project (src/index.ts)

```typescript
import { Project, ProjectAgent } from '@elizaos/core';
import { eliza, spartan, billy, bobby } from './agents';

const elizaAgent: ProjectAgent = {
  character: eliza,
  init: async (runtime) => {
    await setupElizaKnowledge(runtime);
  }
};

const spartanAgent: ProjectAgent = {
  character: spartan,
  init: async (runtime) => {
    await initializeSpartanProtocols(runtime);
  }
};

const billyAgent: ProjectAgent = {
  character: billy,
  init: async (runtime) => {
    await connectBillySystems(runtime);
  }
};

const bobbyAgent: ProjectAgent = {
  character: bobby,
  init: async (runtime) => {
    await setupBobbyCapabilities(runtime);
  }
};

const project: Project = {
  agents: [elizaAgent, spartanAgent, billyAgent, bobbyAgent]
};

export default project;
```

### Standard Project Directory Structure

```
my-project/
├── src/
│   ├── index.ts          # Entry point - exports Project default
│   ├── character.ts      # Character definition
│   ├── __tests__/        # Test files
│   ├── frontend/         # Optional frontend
│   └── plugins/          # Custom project-specific plugins
├── .env                  # Environment variables
├── package.json          # Dependencies
├── tsconfig.json         # TypeScript configuration
├── bun.lock              # Bun lockfile
├── dist/                 # Compiled output
├── node_modules/         # Dependencies
├── .eliza/               # Local runtime data (PGLite, cache, state)
└── scripts/              # Utility scripts
```

### Multi-Agent Project Directory Structure

```
multi-agent-project/
├── src/
│   ├── index.ts
│   ├── agents/
│   │   ├── eliza.ts
│   │   ├── spartan.ts
│   │   ├── billy.ts
│   │   └── bobby.ts
│   ├── __tests__/
│   ├── frontend/
│   └── plugins/
├── .env
├── package.json
├── tsconfig.json
├── dist/
├── node_modules/
├── .eliza/
└── scripts/
```

### CLI Commands

| Command | Description |
|---------|-------------|
| `elizaos start` | Launch project |
| `elizaos dev` | Development mode with hot reload |
| `elizaos start --character ./custom-character.ts` | Start with custom character |
| `elizaos test` | Run all tests |
| `elizaos test --type component` | Component tests only |
| `elizaos test --type e2e` | End-to-end tests |
| `elizaos test --name "pattern"` | Filter tests by name |
| `bun run build` | Manual compilation |

### Project Reset

```bash
rm -rf node_modules dist .eliza
bun install
```

### Per-Agent Model Configuration

```typescript
settings: {
  modelProvider: 'anthropic',  // or 'openai', 'llama', etc.
}
```

### Advanced Project Templates

- **Spartan**: Onchain trading with multi-modal reasoning and memory
- **The Org**: Business swarm with inter-agent communication
- **3D Hyperfy Starter**: WebSocket connection to virtual worlds
- **Next.js Starter**: Web interface with Socket.IO and document management

---

## 6. ENVIRONMENT VARIABLES (https://docs.elizaos.ai/projects/environment-variables + .env.example)

### Complete Environment Variable Reference

#### Server Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `SERVER_PORT` | Port number for the server | 3000 |
| `SERVER_HOST` | Host address to bind to | 0.0.0.0 |
| `NODE_ENV` | Environment mode (development/production) | development |
| `ELIZA_UI_ENABLE` | Web UI toggle (true/false/auto) | Auto (dev=on, prod=off) |
| `ELIZA_SERVER_AUTH_TOKEN` | API auth token for /api/* routes | Unset (no auth) |
| `EXPRESS_MAX_PAYLOAD` | Maximum request payload size | 2mb |

#### Provider Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `PROVIDERS_TOTAL_TIMEOUT_MS` | Timeout for parallel providers | 1000 |

#### Database Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `POSTGRES_URL` | PostgreSQL connection string | - |
| `PGLITE_DATA_DIR` | PGLite database directory (or memory:// for in-memory) | - |

#### AI Model Provider APIs

| Variable | Description |
|----------|-------------|
| `OPENAI_API_KEY` | OpenAI API key |
| `GOOGLE_GENERATIVE_AI_API_KEY` | Google Generative AI key |
| `ANTHROPIC_API_KEY` | Anthropic Claude API key |
| `OPENROUTER_API_KEY` | OpenRouter API key |
| `OLLAMA_API_ENDPOINT` | Local LLM hosting endpoint |

#### Character & Content Loading

| Variable | Description |
|----------|-------------|
| `REMOTE_CHARACTER_URLS` | Comma-separated remote character URLs |

#### Development & Build Control

| Variable | Description |
|----------|-------------|
| `ELIZA_NONINTERACTIVE` | Non-interactive CLI mode toggle |

#### Data Directory Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `ELIZA_DATA_DIR` | Base data directory | .eliza |
| `ELIZA_DATABASE_DIR` | Database directory | - |
| `ELIZA_DATA_DIR_CHARACTERS` | Characters storage directory | - |
| `ELIZA_DATA_DIR_GENERATED` | AI-generated content directory | - |
| `ELIZA_DATA_DIR_UPLOADS_AGENTS` | Agent uploads directory | - |
| `ELIZA_DATA_DIR_UPLOADS_CHANNELS` | Channel uploads directory | - |

#### Plugin Control

| Variable | Description |
|----------|-------------|
| `IGNORE_BOOTSTRAP` | Skip loading the bootstrap plugin |

#### Knowledge Plugin Variables

| Variable | Description |
|----------|-------------|
| `LOAD_DOCS_ON_STARTUP` | Load documents from docs folder on startup |
| `CTX_KNOWLEDGE_ENABLED` | Enable contextual knowledge |
| `KNOWLEDGE_PATH` | Custom path to knowledge documents |
| `MAX_INPUT_TOKENS` | Maximum input tokens for knowledge processing |
| `MAX_OUTPUT_TOKENS` | Maximum output tokens for knowledge responses |

### Authentication Details (ELIZA_SERVER_AUTH_TOKEN)

When set, external applications must send:
```
X-API-KEY: your-secret-token
```
in request headers when calling `/api/*` endpoints.

- OPTIONS requests always allowed (CORS preflight)
- Incorrect/missing key returns 401 Unauthorized
- When unset, all API endpoints are publicly accessible

### UI Control Details (ELIZA_UI_ENABLE)

| Value | Behavior |
|-------|----------|
| `true` | Force enable UI |
| `false` | Force disable UI; non-API routes return 403 Forbidden |
| Unset | Auto: enabled in development, disabled in production |

Dashboard URL only displays on startup when UI is enabled.

### Configuration Examples

**1. Secure Production Deployment:**
```env
NODE_ENV=production
ELIZA_SERVER_AUTH_TOKEN=secure-random-token-here
ELIZA_UI_ENABLE=false
```

**2. Convenient Development Setup:**
```env
NODE_ENV=development
# ELIZA_SERVER_AUTH_TOKEN unset
# ELIZA_UI_ENABLE unset (auto-enabled)
```

**3. Headless API Server:**
```env
ELIZA_SERVER_AUTH_TOKEN=api-only-token
ELIZA_UI_ENABLE=false
```

**4. Public Web Application:**
```env
NODE_ENV=production
ELIZA_SERVER_AUTH_TOKEN=my-api-key
ELIZA_UI_ENABLE=true
```

### Security Recommendations

- Production: Set ELIZA_SERVER_AUTH_TOKEN to strong random values
- Production: Keep ELIZA_UI_ENABLE=false unless web interface is necessary
- Use HTTPS via reverse proxy in production
- Development defaults optimize for convenience over security

---

## 7. INSTALLATION (https://docs.elizaos.ai/installation)

### Prerequisites

| Requirement | Version |
|-------------|---------|
| **Node.js** | 23.3+ |
| **Bun** | Latest |

### Install the CLI

```bash
bun i -g @elizaos/cli
```

This installs the `elizaos` command globally.

### Verify Installation

```bash
elizaos --version
```

### Platform-Specific Instructions

#### macOS / Linux - Node.js via nvm

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install 23.3
nvm use 23.3
```

#### macOS / Linux - Bun

```bash
curl -fsSL https://bun.sh/install | bash
```

#### Windows - Bun

```powershell
powershell -c "irm bun.sh/install.ps1 | iex"
```

#### Windows - Two Paths

1. **WSL2** (Windows Subsystem for Linux) -- recommended
2. **Native Windows** + Git Bash terminal + Git for Windows

Native Windows requires **Git Bash** (not PowerShell or Command Prompt).

### Troubleshooting

| Issue | Solution |
|-------|----------|
| Check Node version | `node --version` |
| Check Bun version | `bun --version` |
| Clear Bun cache | `bun pm cache rm` |
| Add Bun to PATH | `export PATH="$HOME/.bun/bin:$PATH"` (add to ~/.bashrc or ~/.zshrc) |
| Clean CLI reinstall | Remove via all package managers, then `bun i -g @elizaos/cli` |
| Windows Git Bash PATH | `echo 'export PATH=$PATH:"/c/Program Files/nodejs"' >> ~/.bashrc` |

### Important Note

Only clone the monorepo when contributing to core framework development. The CLI handles everything needed for building agents and projects.

---

## 8. QUICKSTART (https://docs.elizaos.ai/quickstart)

### Overview

Launch a functional AI agent with memory and actions capabilities in ~3 minutes. No Docker or complex configuration required.

### Step-by-Step

**Step 1: Create Project**
```bash
elizaos create
```
(The CLI builds agents/projects; the monorepo is only for core contributions.)

**Step 2: Configuration Selections**

During interactive setup, specify:
- **Project name**: Custom name
- **Database**: `pglite` (lightweight local PostgreSQL -- recommended for quickstart)
- **Model provider**: `OpenAI` (for this guide)
- **OpenAI API key**: Obtain from https://platform.openai.com/

Both database and provider selections are reconfigurable later.

**Step 3: Navigate to Project**
```bash
cd my-eliza-project
```

**Step 4: Launch Agent**
```bash
elizaos start
```

**Step 5: Access Web Interface**
```
http://localhost:3000
```

### Environment Configuration

**.env file:**
```env
OPENAI_API_KEY=your-actual-api-key-here
PGLITE_DATA_DIR=/path/to/your/project/.eliza/.elizadb
```

**Alternative setup:**
```bash
elizaos env
```
(Terminal-based environment configuration)

### Troubleshooting

**API Key Issues:**
- Verify `.env` file contains correctly formatted key
- Confirm key format starts with `sk-`
- Check for missing spaces/quotes
- Verify OpenAI account has remaining credits

**Database Connection Issues:**
- Validate `PGLITE_DATA_DIR` path accuracy
- Confirm `.eliza/.elizadb` directory exists with proper permissions
- Recreate project if database corruption occurs

**General Issues:**
- Restart server: `Ctrl+C` then `elizaos start`
- If frontend inaccessible after 10 seconds: `rm -rf my-eliza-project && elizaos create`

### Database Options

| Database | Use Case |
|----------|----------|
| `pglite` | Local development, lightweight, zero-config |
| `postgres` | Production, multi-agent, shared state |

### Model Provider Options

| Provider | Variable |
|----------|----------|
| OpenAI | `OPENAI_API_KEY` |
| Anthropic | `ANTHROPIC_API_KEY` |
| Google | `GOOGLE_GENERATIVE_AI_API_KEY` |
| OpenRouter | `OPENROUTER_API_KEY` |
| Ollama | `OLLAMA_API_ENDPOINT` (local) |

### Next Steps After Quickstart

- Customize agents (personality, knowledge, behavior)
- Create plugins (custom actions, providers, services)
- Deploy to cloud production
- Access 90+ plugins for Discord, Twitter, Solana, and more
