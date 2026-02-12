# Application Integration Patterns

## Two Approaches: Embedded vs External Server

ElizaOS can be integrated into applications in two fundamentally different ways:

### Approach 1: Embedded Runtime (Direct Import)

Import `@elizaos/core` directly into your application and manage `AgentRuntime` instances in-process. Best for monorepos and applications where you control the full stack.

```typescript
import { AgentRuntime, type Character } from '@elizaos/core';

class AgentOrchestrator {
  private agents = new Map<string, AgentRuntime>();

  async startAgent(agentId: string, character: Character): Promise<AgentRuntime> {
    if (this.agents.has(agentId)) return this.agents.get(agentId)!;

    const runtime = new AgentRuntime({
      character,
      plugins: [/* plugin instances */],
      databaseAdapter: myAdapter,
    });
    await runtime.initialize();
    this.agents.set(agentId, runtime);
    return runtime;
  }

  async stopAgent(agentId: string): Promise<void> {
    const runtime = this.agents.get(agentId);
    if (runtime) {
      await runtime.stop();
      this.agents.delete(agentId);
    }
  }

  async sendMessage(agentId: string, text: string, userId: string): Promise<string> {
    const runtime = this.agents.get(agentId);
    if (!runtime) throw new Error('Agent not running');
    // Process message through runtime pipeline
    const response = await runtime.processMessage(/* ... */);
    return response;
  }
}
```

**Pros**: Full control, no network latency, direct access to runtime internals, single deployment.
**Cons**: Agents share process memory, harder to scale independently, plugin conflicts possible.

**Real-world example (ElizaPets)**: Wraps `AgentRuntime` in a Hono API server. Agents are lazy-started on first chat message and auto-stopped after 30min inactivity. Character config is loaded from the database per-agent.

### Approach 2: External Server (REST API)

Run ElizaOS as a standalone server (`elizaos start`) and communicate via its REST API. Best for decoupled architectures, polyglot stacks, or when ElizaOS needs to run independently.

```typescript
const ELIZA_SERVER = process.env.ELIZA_SERVER_URL || 'http://localhost:3000';

// Create an agent
async function createAgent(character: object): Promise<string> {
  const res = await fetch(`${ELIZA_SERVER}/api/agents`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-API-KEY': process.env.ELIZA_SERVER_AUTH_TOKEN,
    },
    body: JSON.stringify(character),
  });
  const agent = await res.json();
  return agent.id;
}

// Start the agent runtime
async function startAgent(agentId: string): Promise<void> {
  await fetch(`${ELIZA_SERVER}/api/agents/${agentId}/start`, { method: 'POST' });
}

// Send a message via DM channel
async function sendMessage(agentId: string, userId: string, text: string): Promise<string> {
  // Get or create DM channel
  const dmRes = await fetch(`${ELIZA_SERVER}/api/messaging/dm/${agentId}/${userId}`, {
    method: 'POST',
  });
  const { channelId } = await dmRes.json();

  // Send message to channel
  const msgRes = await fetch(`${ELIZA_SERVER}/api/messaging/channels/${channelId}/messages`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      content: { text },
      senderId: userId,
      senderName: 'User',
    }),
  });
  return await msgRes.json();
}
```

**Pros**: Language-agnostic, independent scaling, process isolation, can use ElizaOS web dashboard.
**Cons**: Network latency, more complex deployment (two services), less control over runtime internals.

**Real-world example (Agent-Town)**: A Convex-backed game server communicates with ElizaOS via REST. Creates agents with `POST /api/agents`, sends messages with `POST /api/agents/{id}/message`. Falls back to built-in LLM if ElizaOS is unavailable.

### Choosing an Approach

| Factor | Embedded | External Server |
|--------|----------|----------------|
| Latency | Lower (in-process) | Higher (network) |
| Scalability | Single process | Independent scaling |
| Control | Full runtime access | API surface only |
| Deployment | Single service | Two services |
| Language | TypeScript/JS only | Any language |
| Fault isolation | Shared process | Process isolation |
| Best for | Monorepos, tight integration | Microservices, polyglot |

## Dynamic Context Injection

Inject real-time application state into agent prompts so the AI responds contextually. This is the most impactful pattern for making agents feel "aware" of your application.

### Pattern: Prepend Context to Prompts

```typescript
// Build dynamic context from application state
function buildPetContext(pet: Pet): string {
  const lines: string[] = ['[Current game state]'];
  lines.push(`Your name is ${pet.name}, a ${pet.species}.`);
  lines.push(`Your owner has ${pet.neoTokens} NeoTokens.`);

  if (pet.knowledge?.length) {
    lines.push(`You've studied ${pet.knowledge.length} topics and can discuss them knowledgeably.`);
  }

  return lines.join('\n');
}

function buildLocationContext(location: Location, visitorPet: Pet): string {
  const lines: string[] = ['[Visitor context]'];
  lines.push(`A visitor named ${visitorPet.name} (a ${visitorPet.species}) has entered.`);

  if (location.shopItems?.length) {
    lines.push('Your shop currently sells:');
    for (const item of location.shopItems) {
      lines.push(`  - ${item.name}: ${item.price} tokens — ${item.description}`);
    }
    lines.push('Recommend items naturally in conversation when relevant.');
  }

  if (location.cryptoTheme) {
    lines.push(`You specialize in ${location.cryptoTheme}. Share insights when relevant.`);
  }

  return lines.join('\n');
}

// Inject before sending to agent
const context = buildPetContext(pet);
const response = await runtime.processMessage(message, { dynamicContext: context });
```

### Pattern: Provider-Based Context (ElizaOS Native)

For embedded runtimes, use a custom Provider to inject context via the standard ElizaOS pipeline:

```typescript
const gameStateProvider: Provider = {
  name: 'GAME_STATE',
  description: 'Current game state for the player',
  dynamic: true,
  position: -90, // Run early so other providers can reference this data
  get: async (runtime, message, state) => {
    const pet = await getPetForUser(message.entityId);
    if (!pet) return { text: '', values: {}, data: {} };

    return {
      text: [
        `Player's pet: ${pet.name} (${pet.species})`,
        `Token balance: ${pet.neoTokens}`,
        `Known topics: ${pet.knowledgeTopics?.join(', ') || 'none'}`,
        `Current location: ${pet.location || 'wandering'}`,
      ].join('\n'),
      values: { petName: pet.name, tokens: String(pet.neoTokens) },
      data: { pet },
    };
  },
};
```

**Key insight**: The Provider approach integrates with ElizaOS's full state composition pipeline, while the dynamic context approach is simpler and works with any integration pattern (embedded or external).

## Agent Lifecycle in Applications

### Lazy Start / Auto-Stop

Don't keep all agents running permanently. Start them on demand and stop them after inactivity:

```typescript
class AgentOrchestrator {
  private agents = new Map<string, { runtime: AgentRuntime; lastActivity: number }>();
  private cleanupInterval: NodeJS.Timer;

  constructor(private inactivityTimeout = 30 * 60 * 1000) { // 30 minutes
    this.cleanupInterval = setInterval(() => this.cleanup(), 60_000);
  }

  async getOrStartAgent(agentId: string): Promise<AgentRuntime> {
    const existing = this.agents.get(agentId);
    if (existing) {
      existing.lastActivity = Date.now();
      return existing.runtime;
    }

    const config = await loadAgentConfig(agentId);
    const runtime = await this.createRuntime(config);
    this.agents.set(agentId, { runtime, lastActivity: Date.now() });
    return runtime;
  }

  private async cleanup() {
    const now = Date.now();
    for (const [id, agent] of this.agents) {
      if (now - agent.lastActivity > this.inactivityTimeout) {
        await agent.runtime.stop();
        this.agents.delete(id);
      }
    }
  }
}
```

### Config Hot-Reload (Stop + Restart)

When agent configuration changes (e.g., new knowledge learned, personality updated), stop the running agent so it restarts with fresh config on next message:

```typescript
async function updateAgentKnowledge(agentId: string, newKnowledge: string[]) {
  // 1. Update config in database
  const config = await getAgentConfig(agentId);
  config.knowledge = [...(config.knowledge || []), ...newKnowledge];
  await saveAgentConfig(agentId, config);

  // 2. Stop running agent (next message will lazy-restart with new config)
  await orchestrator.stopAgent(agentId);
}
```

### Concurrent Agent Init

ElizaOS has a 30-second timeout for agent initialization. When starting multiple agents:

```typescript
// WRONG — may timeout or cause race conditions
await Promise.all(agentIds.map(id => orchestrator.startAgent(id)));

// RIGHT — stagger startup with delays
for (const id of agentIds) {
  await orchestrator.startAgent(id);
  await new Promise(r => setTimeout(r, 2000)); // 2s delay between agents
}
```

## Dual AI Architecture (Fallback Pattern)

When ElizaOS availability isn't guaranteed, implement a fallback to direct LLM calls:

```typescript
async function getAgentResponse(agentId: string, message: string): Promise<string> {
  // Try ElizaOS first
  try {
    const runtime = await orchestrator.getOrStartAgent(agentId);
    const response = await runtime.processMessage(message);
    if (response) return response;
  } catch (error) {
    console.warn('ElizaOS unavailable, falling back to direct LLM:', error.message);
  }

  // Fallback: direct LLM call with character context
  const character = await loadCharacterConfig(agentId);
  const systemPrompt = buildSystemPrompt(character);

  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-5-20250929',
    system: systemPrompt,
    messages: [{ role: 'user', content: message }],
  });

  return response.content[0].text;
}

function buildSystemPrompt(character: Character): string {
  const parts: string[] = [];
  parts.push(`You are ${character.name}.`);
  if (character.bio?.length) parts.push(character.bio.join(' '));
  if (character.style?.all?.length) parts.push('Style: ' + character.style.all.join('. '));
  if (character.knowledge?.length) {
    parts.push('You know about: ' + character.knowledge.filter(k => typeof k === 'string').join(', '));
  }
  return parts.join('\n\n');
}
```

**Tradeoff**: The fallback loses ElizaOS memory, evaluators, actions, and multi-step planning. It's useful for availability but should be clearly marked as degraded mode.

## Game / Simulation Integration

### Autonomous Agent Loop (Tick-Based)

For games where agents act independently (not just responding to player messages):

```typescript
interface AgentState {
  agentId: string;
  position: { x: number; y: number };
  currentActivity: string;
  conversationPartner: string | null;
  lastDecisionTime: number;
}

async function agentTick(state: AgentState, worldState: WorldState): Promise<AgentState> {
  const now = Date.now();

  // Only make decisions every N seconds
  if (now - state.lastDecisionTime < 5000) return state;

  // Build context from world state
  const context = [
    `You are at position (${state.position.x}, ${state.position.y}).`,
    `Nearby agents: ${worldState.nearbyAgents.map(a => a.name).join(', ')}`,
    `Current activity: ${state.currentActivity}`,
    `Available activities: ${worldState.availableActivities.join(', ')}`,
  ].join('\n');

  // Ask ElizaOS agent to decide next action
  const decision = await runtime.useModel(ModelType.TEXT_SMALL, {
    prompt: `${context}\n\nWhat should you do next? Choose: continue, move, talk, or new_activity.`,
    temperature: 0.8,
  });

  // Parse decision and update state
  return applyDecision(state, decision);
}
```

### Multi-Agent Conversation Routing

When multiple agents exist in the same world and can converse with each other:

```typescript
async function routeConversation(
  initiatorId: string,
  targetId: string,
  message: string,
): Promise<{ response: string; shouldContinue: boolean }> {
  // Get both agent runtimes
  const initiator = await orchestrator.getOrStartAgent(initiatorId);
  const target = await orchestrator.getOrStartAgent(targetId);

  // Send message to target agent with initiator context
  const context = `${initiator.character.name} says to you: "${message}"`;
  const response = await target.processMessage(context, {
    dynamicContext: `You are in conversation with ${initiator.character.name}.`,
  });

  // Evaluate if conversation should continue
  const shouldContinue = await evaluateConversationEnergy(response);

  return { response, shouldContinue };
}
```

### Knowledge Learning System

Enable agents to acquire new knowledge at runtime through in-game mechanics (books, scrolls, items):

```typescript
interface KnowledgeItem {
  id: string;
  name: string;
  knowledgeEntries: string[];  // Strings added to character.knowledge[]
  price: number;
}

async function learnFromItem(agentId: string, item: KnowledgeItem): Promise<void> {
  // 1. Load current character config
  const config = await loadCharacterConfig(agentId);
  const currentKnowledge: string[] = config.knowledge ?? [];

  // 2. Merge new entries (deduplicate)
  const newEntries = item.knowledgeEntries.filter(e => !currentKnowledge.includes(e));
  config.knowledge = [...currentKnowledge, ...newEntries];

  // 3. Persist updated config
  await saveCharacterConfig(agentId, config);

  // 4. Stop agent so next interaction uses updated config
  await orchestrator.stopAgent(agentId);
}
```

**Key insight**: ElizaOS character `knowledge` is loaded at agent startup. To make an agent "learn" new things, update the config in your database and restart the agent. The next `processMessage` call will lazy-start with the new knowledge.

## Memory Augmentation Patterns

ElizaOS has built-in vector memory (5 types). Applications can layer additional memory patterns on top:

### Importance Scoring

Not all memories are equal. Score them so retrieval prioritizes significant events:

```typescript
async function scoreMemoryImportance(text: string): Promise<number> {
  const response = await runtime.useModel(ModelType.TEXT_SMALL, {
    prompt: `Rate the importance of this memory on a scale of 0-9, where 0 is mundane (e.g., routine greeting) and 9 is critical (e.g., learned a major secret, completed a quest).

Memory: "${text}"

Respond with just the number.`,
  });
  return Math.min(9, Math.max(0, parseInt(response) || 3));
}

// Store with importance metadata
await runtime.createMemory({
  type: MemoryType.CUSTOM,
  content: { text: memoryText },
  metadata: { importance: await scoreMemoryImportance(memoryText), source: 'conversation' },
  roomId, entityId,
});
```

### Reflection (Meta-Cognitive Summaries)

Periodically synthesize recent memories into higher-level insights:

```typescript
async function reflectOnRecentMemories(runtime: IAgentRuntime, threshold: number = 100) {
  // Get recent memories with importance scores
  const recentMemories = await runtime.searchMemories({
    type: MemoryType.CUSTOM,
    limit: 50,
  });

  // Calculate accumulated importance
  const totalImportance = recentMemories.reduce(
    (sum, m) => sum + ((m.metadata as any)?.importance ?? 3), 0
  );

  if (totalImportance < threshold) return; // Not enough significant memories yet

  // Generate reflection
  const memoryTexts = recentMemories.map(m => `- ${m.content.text}`).join('\n');
  const reflection = await runtime.useModel(ModelType.TEXT_LARGE, {
    prompt: `Given these recent memories:\n${memoryTexts}\n\nWhat are 3 high-level insights or patterns you notice? Be specific and actionable.`,
  });

  // Store reflection as a high-importance memory
  await runtime.createMemory({
    type: MemoryType.CUSTOM,
    content: { text: `[Reflection] ${reflection}` },
    metadata: { importance: 8, source: 'reflection', isReflection: true },
  });
}
```

### Recency-Weighted Retrieval

Combine vector similarity with recency decay for more relevant memory retrieval:

```typescript
function scoreMemoryRelevance(
  memory: Memory,
  similarityScore: number,
  now: number = Date.now()
): number {
  const ageHours = (now - new Date(memory.createdAt).getTime()) / (1000 * 60 * 60);
  const recencyDecay = Math.exp(-ageHours / 168); // Half-life of ~1 week
  const importance = (memory.metadata as any)?.importance ?? 3;
  const importanceWeight = importance / 9;

  // Weighted combination: 50% similarity, 30% recency, 20% importance
  return (similarityScore * 0.5) + (recencyDecay * 0.3) + (importanceWeight * 0.2);
}
```

## Token Economy Integration

When building applications with in-game currency that agents should be aware of:

```typescript
// Inject token balance into agent context
function buildEconomyContext(pet: { neoTokens: number; inventory: Item[] }): string {
  const lines = [];
  lines.push(`Token balance: ${pet.neoTokens} NeoTokens`);

  if (pet.inventory.length > 0) {
    lines.push(`Inventory: ${pet.inventory.map(i => i.name).join(', ')}`);
  }

  return lines.join('\n');
}

// Award tokens through gameplay (e.g., chatting with location NPCs)
async function awardTokensForInteraction(petId: string, amount: number): Promise<void> {
  await db.update(pets)
    .set({ neoTokens: sql`neo_tokens + ${amount}` })
    .where(eq(pets.id, petId));
}
```

## Deployment Patterns

### Embedded Runtime on Railway/Render

Single service running both your API and ElizaOS agents:

```dockerfile
FROM oven/bun:1
WORKDIR /app
COPY . .
RUN bun install
RUN bun run build
ENV NODE_ENV=production
ENV POSTGRES_URL=... # Set via Railway/Render dashboard
EXPOSE 4000
CMD ["bun", "run", "src/index.ts"]
```

**Key env vars**: `DATABASE_URL`, `POSTGRES_URL` (ElizaOS needs this specifically), `ANTHROPIC_API_KEY`, `OPENAI_API_KEY` (for embeddings).

### External ElizaOS Server + Separate App

```yaml
# docker-compose.yml
services:
  elizaos:
    image: elizaos/eliza:latest
    environment:
      - POSTGRES_URL=${POSTGRES_URL}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - ELIZA_SERVER_AUTH_TOKEN=${ELIZA_AUTH_TOKEN}
    ports:
      - "3000:3000"

  app:
    build: .
    environment:
      - ELIZA_SERVER_URL=http://elizaos:3000
      - ELIZA_AUTH_TOKEN=${ELIZA_AUTH_TOKEN}
    ports:
      - "4000:4000"
    depends_on:
      - elizaos
```

### Critical Deployment Notes

- **ElizaOS plugin-sql creates its own tables** (agents, memories, rooms, entities, etc.) via auto-migration. Don't pre-create these tables or their indexes — let ElizaOS handle it. Pre-creating indexes causes migration failures.
- **`POSTGRES_URL` is required** alongside any other database URL your app uses. ElizaOS specifically looks for `POSTGRES_URL`.
- **Concurrent agent startup**: Stagger agent creation with 2-5 second delays. ElizaOS has a 30-second init timeout.
- **Memory usage**: Each running agent consumes ~50-100MB. Plan capacity accordingly.
- **Serverless doesn't work**: ElizaOS agents need persistent processes. Don't deploy on Vercel/Netlify Functions for the agent runtime. Use Railway, Render, Fly.io, or similar.
