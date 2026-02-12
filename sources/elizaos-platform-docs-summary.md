# ElizaOS Comprehensive Technical Reference
## Extracted from docs.elizaos.ai (20 documentation pages)

---

## 1. DISCORD PLUGIN (@elizaos/plugin-discord)

### Architecture
- **DiscordService**: Connection management, client init with Discord.js, event routing, channel access controls via `CHANNEL_IDS` env var
- **MessageManager**: Message processing pipeline (reception/filtering -> format conversion -> attachment processing -> response handling)
- **VoiceManager**: Voice channel ops (connection lifecycle, audio capture, transcription, speaker detection)

### Event Processing Pipeline
1. Bot message filtering (ignores self-generated)
2. Channel restriction checking
3. Attachment processing (images via vision analysis, audio via transcription, videos, documents)
4. elizaOS format conversion with Discord context metadata
5. Includes: message ID, timestamp, edited timestamp, pin status, mention tracking

### Event Types
- **MESSAGE_CREATE**: Permissions check -> MessageManager -> conversation context tracking
- **INTERACTION_CREATE**: Routes slash commands by command name
- **GUILD_CREATE**: Server room creation, emits `WORLD_JOINED`, registers slash commands per-server
- **GUILD_MEMBER_ADD**: Entity creation
- **VOICE_STATE_UPDATE**: Connection management

### Interaction Routing
- Slash commands: validate permissions, parse args, execute actions
- Button handlers: process component interactions
- Select menu handlers: route menu selections

### Actions
- **CHAT_WITH_ATTACHMENTS**: Processes media-inclusive messages, extracts content, generates contextual responses
- **JOIN_VOICE**: Connects bot to voice channels with channel type validation
- **TRANSCRIBE_MEDIA**: Converts audio/video URLs to text transcripts

### Providers
- **CHANNEL_STATE**: Channel metadata (name, type, guild info, member count)
- **VOICE_STATE**: Active voice info, member speaking status, connection health

### Configuration
```
# Required
DISCORD_APPLICATION_ID
DISCORD_API_TOKEN

# Optional
CHANNEL_IDS                  # Comma-separated channel IDs for restrictions
DISCORD_VOICE_CHANNEL_ID
VOICE_ACTIVITY_THRESHOLD
DISCORD_TEST_CHANNEL_ID
```

### Permissions Required
- Text: SendMessages, EmbedLinks, ReadMessageHistory
- Voice: Connect, Speak, UseVAD
- Commands: UseApplicationCommands
- OAuth2 URL: `client_id` + calculated `permissions` bitfield + `scope=bot%20applications.commands`

### Multi-Server Architecture
- Each server: isolated conversation maps, user relationships, channel states, voice connections
- Slash commands register per-guild (not globally)

### Performance
- Message caching: LRU with 1-hour TTL
- Rate limiting: 30 requests/minute windows
- Voice connection pooling: reuse existing connections matching guild IDs with "Ready" status
- Event batching: 100ms collection window
- Max 10 concurrent voice connections with LRU eviction
- Per-handler try-catch prevents cascading failures

### Error Handling
- Connection errors trigger reconnection scheduling
- API error codes: 10008 (unknown message), 50001 (access denied), 50013 (permission failure)
- Permission failures attempt graceful notification in accessible channels

### Voice Flow
- Join events create VoiceConnection with audio stream setup
- Speaking state triggers transcription processing
- Transcribed audio re-enters standard message pipeline
- Leave events cleanup resources and disconnect

### State Interfaces
- **MessageContext**: channelId, serverId, userId, threadId, referencedMessageId, attachments, discordMetadata
- **VoiceContext**: channelId, serverId, VoiceConnection instance, activeUsers mapping, recordingState with buffer management

### Monitoring
- EventMetrics: type, processingTime, success, errorType, serverId, channelId
- Slow events (>1000ms) trigger warnings

---

## 2. TELEGRAM PLUGIN (@elizaos/plugin-telegram)

### Architecture
- **TelegramService**: Bot lifecycle, Telegraf client init, middleware, event handlers, chat sync via `knownChats`/`syncedEntityIds` maps
- **MessageManager**: Message conversion to elizaOS format, media processing, callback management

### Configuration
```
# Required
TELEGRAM_BOT_TOKEN=your_token_here

# Optional
TELEGRAM_API_ROOT            # Custom API endpoint
TELEGRAM_ALLOWED_CHATS       # JSON array of permitted chat IDs
TELEGRAM_TEST_CHAT_ID        # Integration testing

# Character config options
allowDirectMessages          # DM support toggle
shouldOnlyJoinInAllowedGroups # Group restrictions
messageTrackingLimit         # Conversation history size
```

### Message Processing Pipeline
```
Message Update -> Telegraf Middleware -> Chat/User Sync -> Message Manager -> Bootstrap Plugin -> Response Formatting -> Telegram API
```

### Validation Gates
- Direct message allowance checks
- Group whitelist verification
- Forum topic detection and context retrieval

### Message Types
- **Text**: Direct `ctx.message?.text` extraction
- **Photos**: Highest-resolution download via `bot.telegram.getFile()`, vision analysis
- **Voice**: Transcription via `runtime.transcribe(audioBuffer)` with MIME type and duration
- **Documents**: Route by MIME type (images -> image processing, text -> content extraction, other -> generic attachment)

### Message Context Structure
- chatId, chatType (private/group/supergroup/channel)
- messageId, userId, optional username
- threadId for forum topics
- replyToMessageId for replies

### Interactive Elements
```typescript
// Inline keyboards
{ text: "Option 1", callback_data: "opt_1" }
{ text: "Link", url: "https://example.com" }

// Callback handling
ctx.callbackQuery.data  // capture data
ctx.answerCbQuery()     // remove loading state

// Reply keyboards
{ resize_keyboard: true, one_time_keyboard: true }
```

### Forum Topics
- Topics use `message_thread_id` for separate conversation contexts
- Room ID format: `${chatId}-topic-${topicId}` for topics, `chatId.toString()` otherwise
- TopicManager maintains separate histories and metadata per topic key

### Group Features
- Access control checks `shouldOnlyJoinInAllowedGroups` + validates chat ID against allowlist
- Bot admin verification: `ctx.getChatMember(botId)` with status checks
- Privacy mode: disabled = all messages; enabled = mentions, replies to bot, commands only

### Webhook Setup (Production)
```typescript
await bot.telegram.setWebhook('https://domain.com/webhook', {
  certificate: fs.readFileSync('cert.pem'),
  allowed_updates: ['message', 'callback_query'],
  drop_pending_updates: true
});
app.post('/telegram-webhook', (req, res) => {
  bot.handleUpdate(req.body);
  res.sendStatus(200);
});
```

### Error Handling
- 429: Rate limited - exponential backoff with `retry_after`
- 400: Bad request - don't retry
- Network timeouts: retry with backoff
- Connection: exponential backoff, max 5 reconnects, cap at 30s delay
- Error 409: Multiple getUpdates (multi-agent conflict) - use different tokens or webhooks

### State Interfaces
- **TelegramMessageState**: message ID, chat ID, user ID, content, reply-to, thread IDs, entities
- **TelegramChatState**: chat config, forum topic maps, message history
- **CallbackState**: callback query metadata for button interactions

### Response Generation
- Text content, inline buttons, media attachments evaluated
- Long messages auto-split
- Buttons create inline keyboards with callback/URL action types
- Media sent by type (photo/document/audio)

### Performance
- Batching: 100ms windows for parallel processing
- Caching: user, chat, media, response data with TTL
- Webhooks preferred over polling in production

---

## 3. TWITTER PLUGIN (@elizaos/plugin-twitter)

### Architecture
- **TwitterService**: Creates/manages TwitterClientInstance objects, client lifecycle, emits WORLD_JOINED
- **ClientBase**: API init, credential verification
- **TwitterPostClient**: Autonomous tweet generation with scheduling
- **TwitterInteractionClient**: Timeline monitoring, response handling
- **TwitterTimelineClient**: Action evaluation (likes, retweets, quotes)

### Authentication (CRITICAL)
```
# OAuth 1.0a REQUIRED (NOT OAuth 2.0!)
TWITTER_API_KEY              # Consumer API Key
TWITTER_API_SECRET_KEY       # Consumer API Secret
TWITTER_ACCESS_TOKEN         # Access Token (write-enabled)
TWITTER_ACCESS_TOKEN_SECRET  # Access Token Secret
```
After changing permissions, MUST regenerate tokens via Developer Portal.

### Configuration Parameters
```
# Post Generation
TWITTER_POST_ENABLE=false          # Activate autonomous posting
TWITTER_POST_INTERVAL_MIN          # Min interval (minutes)
TWITTER_POST_INTERVAL_MAX          # Max interval (minutes)
TWITTER_POST_INTERVAL_VARIANCE     # Randomization (0.0-1.0)
TWITTER_POST_IMMEDIATELY           # Post on startup
TWITTER_MAX_TWEET_LENGTH=4000      # Max chars

# Interaction Settings
TWITTER_SEARCH_ENABLE=true         # Timeline monitoring
TWITTER_AUTO_RESPOND_MENTIONS      # Auto-reply mentions
TWITTER_AUTO_RESPOND_REPLIES       # Auto-reply replies
TWITTER_MAX_INTERACTIONS_PER_RUN   # Batch limit
TWITTER_INTERACTION_INTERVAL_MIN/MAX # Monitoring frequency

# Timeline Algorithm
TWITTER_TIMELINE_ALGORITHM         # "weighted" or "latest"
TWITTER_TIMELINE_USER_BASED_WEIGHT=3
TWITTER_TIMELINE_TIME_BASED_WEIGHT=2
TWITTER_TIMELINE_RELEVANCE_WEIGHT=5

# Advanced
TWITTER_DRY_RUN                    # Test mode, no posting
TWITTER_TARGET_USERS               # Comma-separated usernames or "*"
TWITTER_RETRY_LIMIT=5
TWITTER_POLL_INTERVAL=120          # Timeline polling seconds
TWITTER_ENABLE_ACTION_PROCESSING=false # Enable likes/retweets
```

### Weighted Timeline Algorithm
- User Score: target users (max 10), verified (+2), high followers (+1)
- Time Score: `10 - (age_in_hours/2)`, capped 0-10
- Relevance Score: content matching via LLM
- Formula: `(userScore * userWeight) + (timeScore * timeWeight) + (relevanceScore * relevanceWeight)`

### Timeline Processing Pipeline
```
Fetch -> Filter -> Score -> Sort -> Process
```
- Cache layers and rate limiters across all stages
- Cache validity check first; if expired, queue through rate-limit before API call

### Interaction Decision Logic
- Direct replies to agent: approve
- Mentions: approve
- Target user tweets: approve if score > threshold
- Other tweets: 30% probability for high-scoring content
- Thread context: relevance determines approval

### Request Management
- **RequestQueue**: FIFO processing, rate limit checks, exponential backoff on 429, pauses during rate windows, 100ms delays between requests
- **RateLimiter**: Per-endpoint sliding windows, detects expired windows, calculates wait times

### Tweet Generation
1. Build context from recent tweets + trending topics
2. LLM generates via `buildTweetPrompt()`
3. Validate with `validateTweet()`
4. Clean and length-verify

### Response Generation
1. Retrieve full thread via `getConversationThread()`
2. Map as alternating user/assistant messages
3. LLM generates contextual response (max 100 tokens)

### Error Handling
- 429: Extract resetTime, pause queue
- 401: Log credential failure, no retry
- 403: Log permission issue, no retry
- 400: Log validation error, no retry
- Network (ECONNRESET, ETIMEDOUT): Retry with exponential backoff (base 1000ms, 2^attempt)
- Max retries: 3 (configurable)

### Performance
- Tweet cache: LRU, max 10,000 items, 1-hour TTL
- User cache: LRU, max 5,000 items, 24-hour TTL
- Timeline cache: 1-minute expiry
- Max 10,000 processed tweet IDs; hourly cleanup
- Batch operations via `client.v2.users(userIds)` / `client.v2.tweets(tweetIds)` + `Promise.all()`

### Dry Run / Multi-Account
- `simulateTweet()` returns synthetic objects without API calls
- Multi-account: `twitterService.createClient(runtime, clientId, config)` - each client has separate state

---

## 4. FARCASTER PLUGIN (@elizaos/plugin-farcaster)

### Setup
```bash
npm install @elizaos/plugin-farcaster
```

### Configuration
```
# Required
FARCASTER_NEYNAR_API_KEY     # Neynar authentication
FARCASTER_SIGNER_UUID        # Message signing
FARCASTER_FID                # Agent's Farcaster ID (numeric)

# Timing (minutes)
CAST_INTERVAL_MIN=90
CAST_INTERVAL_MAX=180
FARCASTER_POLL_INTERVAL=2
ACTION_INTERVAL=5

# Feature Toggles
ENABLE_CAST=true
ENABLE_ACTION_PROCESSING=false
FARCASTER_DRY_RUN=false
CAST_IMMEDIATELY=false

# Constraints
MAX_CAST_LENGTH=320
MAX_ACTIONS_PROCESSING=1
ACTION_TIMELINE_TYPE=ForYou   # or Following

# Debug
FARCASTER_DEBUG=true
LOG_LEVEL=debug
```

### Character Configuration
```typescript
export const character: Character = {
  plugins: [farcasterPlugin],
  settings: {
    farcaster: {
      channels: ["/elizaos", "/ai16z"],
      replyProbability: 0.7,
      castStyle: "conversational",
      maxCastLength: 320
    }
  }
};
```

### Actions & Providers
- **SEND_CAST**: Post original messages
- **REPLY_TO_CAST**: Threaded replies
- **farcasterProfile** provider: FID, username, bio, followers
- **farcasterTimeline** provider: Recent casts for context

### Service Architecture
- **FarcasterService**: Main orchestrator with `initialize()`, `start()`, `stop()`, `healthCheck()`, manages agents via UUID
- **MessageService**: `getMessages()`, `getMessage()`, `sendMessage()`
- **CastService** (IPostService): `createCast()`, `deleteCast()`, `getCasts()`, `likeCast()`, `recast()`, `unlikeCast()`, `unrecast()`, `getProfile(fid)`, `getCastByHash(hash)`

### Client Operations (FarcasterClient)
```typescript
// Publishing
publishCast(text, { embeds?, replyTo?, channelId? })
reply({ text, replyTo: { hash, fid } })
deleteCast(targetHash)

// User Data
getUser() / getUserByFid(fid) / getUserByUsername(username)

// Timeline & Mentions
getMentions(fid, cursor?)
getTimeline('ForYou' | 'Following', cursor?)
getCast(hash)

// Engagement
likeCast/unlikeCast(targetHash)
recast/unrecast(targetHash)
followUser/unfollowUser(targetFid)
```

### Cast Flow Pipeline (7 stages)
1. Event Reception: Neynar API polling
2. Event Classification: Direct Mention, Channel Cast, Reply Thread, Timeline Cast
3. Content Analysis: tokenization, sentiment, topic extraction, relevance scoring
4. Response Decision: mention status, reply context, keyword matching, sentiment
5. Response Generation: LLM with conversation history + topics
6. Cast Composition: text validation, length check (320), embed validation, metadata
7. Publishing: Neynar API with retry + exponential backoff

### Event Handler Priority
- Direct Mention: priority queue, immediate processing
- Reply Thread: conversation context preservation
- Channel Cast: relevance evaluation before participation
- Timeline Cast: engagement opportunity monitoring

### Memory & Caching
- `cast:sent` stores hash, threadId, messageId, platform
- `message:received` creates cast + profile memories
- LRU Cast cache: 30min TTL, 9000 max entries
- Profile cache, last cast timestamps per-agent

### Utility Functions
```typescript
castUuid(cast): UUID
neynarCastToCast(cast)    // Format conversion
formatCast(cast): string  // AI-ready formatting
formatTimeline(casts[]): string
lastCastCacheKey(agentId): string
```

### Performance
- AsyncQueue with configurable concurrency
- Single-threaded prevents rate-limit collisions
- ~550ms median end-to-end processing

---

## 5. BOOTSTRAP PLUGIN (@elizaos/plugin-bootstrap)

### Role
Core message handler for ALL platforms. Handles message processing, response determination, context composition, action execution, evaluator processing, and state management.

### Event Handlers
| Event | Handler | Function |
|-------|---------|----------|
| MESSAGE_RECEIVED | messageReceivedHandler | Primary message processor |
| VOICE_MESSAGE_RECEIVED | messageReceivedHandler | Voice input handling |
| REACTION_RECEIVED | reactionReceivedHandler | Reaction storage |
| MESSAGE_DELETED | messageDeletedHandler | Deletion tracking |
| CHANNEL_CLEARED | channelClearedHandler | Channel purge handling |
| POST_GENERATED | postGeneratedHandler | Social media creation |
| WORLD_JOINED | handleServerSync | Environment synchronization |
| ENTITY_JOINED | syncSingleUser | User data synchronization |

### Actions (13 Total)
**Response**: REPLY, IGNORE, NONE
**Room Management**: FOLLOW_ROOM, UNFOLLOW_ROOM, MUTE_ROOM, UNMUTE_ROOM
**Advanced**: SEND_MESSAGE, UPDATE_CONTACT, CHOOSE_OPTION, UPDATE_ROLE, UPDATE_SETTINGS, GENERATE_IMAGE

### Providers (17 Total)
RECENT_MESSAGES, TIME, CHARACTER, ENTITIES, RELATIONSHIPS, WORLD, ANXIETY, ATTACHMENTS, CAPABILITIES, ACTIONS, PROVIDERS, EVALUATORS, SETTINGS, ROLES, CHOICE, ACTION_STATE, FACTS

### REFLECTION Evaluator
Post-interaction cognitive processing:
- Conversation quality analysis
- Fact extraction + knowledge base updates
- Relationship identification
- Output: JSON with `thought`, `facts[]` (claim/type/bio_status), `relationships[]` (entity IDs, interaction tags)

### TaskService
- Repeating task execution at intervals
- One-time task processing
- Immediate task execution
- Conditional validation rules

### Handler Pattern
```typescript
{
  name: 'ACTION_NAME',
  similes: ['ALTERNATIVE_NAME'],
  description: 'Functionality description',
  validate: async (runtime) => boolean,
  handler: async (runtime, message, state, options, callback, responses?) => boolean,
  examples: ActionExample[][]
}
```

### Provider Pattern
```typescript
{
  name: 'PROVIDER_NAME',
  description: 'Context provision',
  position: 100,
  get: async (runtime, message) => ({
    data: {},
    values: {},
    text: ''
  })
}
```

### Message Processing Pipeline (Bootstrap)
```
MESSAGE_RECEIVED -> Self-Check -> ID Generation -> Run Tracking
-> Memory Storage (Parallel) -> Attachment Processing -> State Evaluation
-> Mute Status Check -> shouldRespond Decision -> Response Generation
-> Validation Loop (max 3 retries) -> Action Processing -> Evaluator Execution -> RUN_ENDED
```

### Response Bypass Logic
- Direct messages, voice DMs, API calls skip shouldRespond validation
- Configurable via `SHOULD_RESPOND_BYPASS_TYPES` and `SHOULD_RESPOND_BYPASS_SOURCES`

### State Composition
Providers inject context during composition:
- RECENT_MESSAGES, CHARACTER, ENTITIES, TIME, RELATIONSHIPS, WORLD
- All providers execute in parallel; token estimation post-composition

### Configuration
```
SHOULD_RESPOND_BYPASS_TYPES    # Room types bypassing response checks
SHOULD_RESPOND_BYPASS_SOURCES  # Message sources bypassing checks
CONVERSATION_LENGTH=20         # Context window size
RESPONSE_TIMEOUT=3600000       # LLM timeout (ms, default 1 hour)
```

### Template System
Templates use mustache syntax:
- `{{agentName}}`, `{{providers}}`, `{{actionNames}}`, `{{recentMessages}}`
- Output expects XML/JSON parsing for thought, actions, text, provider selections

### Four Primary Templates
1. **shouldRespondTemplate**: Engagement triggers and silence conditions
2. **messageHandlerTemplate**: Response formulation, action selection, personality
3. **reflectionTemplate**: Learning extraction, fact identification, relationship updates
4. **postCreationTemplate**: Social media content generation

### Callback Mechanism
```typescript
callback({
  text: 'Response content',
  actions: ['ACTION_TAG'],
  thought: 'Agent reasoning',       // optional
  attachments: [...],               // optional
  metadata: {...}                   // optional
})
```
Multiple invocations = multiple messages. Omitting callback = no response.

### Platform Abstraction
- Discord: channels -> rooms, servers -> worlds, users -> entities
- Telegram: chats -> rooms, groups -> worlds, users -> entities
- Message Bus: topics -> rooms, namespaces -> worlds, publishers -> entities

---

## 6. SQL DATABASE ADAPTERS (@elizaos/plugin-sql)

### Adapter Implementations

#### PGLite Adapter (Development)
- Embedded PostgreSQL in Node.js, no external deps
- File-based persistence (default: `./.eliza/.elizadb`)
- Singleton connection per process
- Auto-init on first use

```typescript
export class PgliteDatabaseAdapter extends BaseDrizzleAdapter {
  private manager: PGliteClientManager;
  protected embeddingDimension: EmbeddingDimensionColumn = DIMENSION_MAP[384];
  constructor(agentId: UUID, manager: PGliteClientManager)
}
```

#### PostgreSQL Adapter (Production)
- Full PostgreSQL with connection pooling (default: 20 connections)
- SSL support, auto-retry (3 attempts with exponential backoff)
- Compatible with Supabase, Neon, RDS, Cloud SQL

```typescript
export class PgDatabaseAdapter extends BaseDrizzleAdapter {
  private manager: PostgresConnectionManager;
  constructor(agentId: UUID, manager: PostgresConnectionManager, _schema?: any)
  protected async withDatabase<T>(operation: () => Promise<T>): Promise<T>
}
```

### Adapter Selection
```typescript
if (config.postgresUrl) {
  return new PgDatabaseAdapter(agentId, postgresConnectionManager);
} else {
  return new PgliteDatabaseAdapter(agentId, pgLiteClientManager);
}
```

### Environment Variables
```
POSTGRES_URL       # Triggers PostgreSQL adapter
PGLITE_DATA_DIR    # Custom PGLite directory (default: ./.eliza/.elizadb)
```

### Migration
- Handled by `DatabaseMigrationService` (not adapters directly)
- Plugin schema discovery and registration
- Dynamic table creation/updates
- Schema introspection for existing tables
- Dependency resolution for creation order

### Error Handling
- 3 retry attempts with exponential backoff
- Handles: connection timeouts, transient failures, pool exhaustion, filesystem errors

---

## 7. SQL PLUGIN TABLES

### Table Definition (Drizzle ORM)
Tables defined using Drizzle, exported via `schema` property from plugin entry point, auto-created at startup.

### Schema Namespacing
- `@company/my-plugin` becomes schema `company_my_plugin`
- Tables: `company_my_plugin.plugin_users`

### Common Patterns
- Primary key: UUID with random generation
- JSONB for flexible data (profiles, metadata)
- Timestamps with timezone + automatic defaults
- Indexes on agentId, lookup fields, createdAt
- Foreign keys with cascade delete, set null, or restrict
- Composite primary keys for junction tables
- Check constraints for business logic
- Generated columns for computed values (order totals)

### Cross-Plugin References
```typescript
// Fully qualified foreign key
"other_plugin"."users"("id")
```

---

## 8. KNOWLEDGE PLUGIN (@elizaos/plugin-knowledge)

### Setup
- Add plugin + AI provider (OpenAI/OpenRouter) to character config
- Requires `OPENAI_API_KEY` for embeddings (even with other providers)

### Document Support
- Text: `.txt`, `.md`, `.csv`, `.json`, `.xml`, `.yaml`
- Documents: `.pdf`, `.doc`, `.docx`
- Code: `.js`, `.ts`, `.py`, `.java`, `.cpp`, `.html`, `.css`

### Auto-Loading
- Create `docs/` folder in project root
- Set `LOAD_DOCS_ON_STARTUP=true`
- Documents processed into searchable chunks in agent's DB

### Actions
- **PROCESS_KNOWLEDGE**: Document ingestion (file paths or text input)
- **SEARCH_KNOWLEDGE**: Topic-based retrieval from knowledge base

### Configuration
```
OPENAI_API_KEY               # Required for embeddings
OPENROUTER_EMBEDDING_MODEL   # e.g., openai/text-embedding-3-large
LOAD_DOCS_ON_STARTUP=true
KNOWLEDGE_PATH               # Custom document location
```

### Web Interface
- `http://localhost:3000` after `elizaos start`
- Upload, search, deletion functions

---

## 9. CLI REFERENCE (@elizaos/cli)

### Installation
```bash
bun install -g @elizaos/cli
```

### Core Commands
| Command | Purpose |
|---------|---------|
| `create` | Initialize project, plugin, or agent |
| `monorepo` | Clone elizaOS monorepo (default: develop branch) |
| `plugins` | Manage plugins |
| `agent` | Manage agents |
| `tee` | TEE deployments |
| `start` | Launch agent |
| `update` | Update CLI + dependencies |
| `test` | Run tests |
| `env` | Manage env vars/secrets |
| `dev` | Hot-reload development mode |
| `publish` | Publish plugin to registry |
| `deploy` | Deploy to AWS ECS |
| `login` | ElizaOS Cloud auth |
| `containers` | Cloud container management |
| `scenario` | Test scenarios |
| `report` | Scenario matrix reports |

### Global Options
```
--help, -h         # Display help
--version, -v      # Display version
--no-emoji         # Disable emoji output
--no-auto-install  # Skip Bun installation prompt
-d, --debug        # LOG_LEVEL=debug
--verbose          # LOG_LEVEL=trace
-q, --quiet        # LOG_LEVEL=error
--log-json         # JSON log output
```

### Common Workflows
```bash
# Create and start
elizaos create my-agent-project
cd my-agent-project
elizaos start

# Development
elizaos dev                    # Hot-reload
elizaos env edit-local         # Configure vars
elizaos env list               # Display all vars
elizaos env interactive        # Interactive manager

# Plugins
elizaos plugins list
elizaos plugins add @elizaos/plugin-discord
elizaos publish                # From plugin directory
elizaos publish --test         # Test mode

# Testing
elizaos test
elizaos test --type component
elizaos test --type e2e
elizaos test --name "pattern"

# Logging
elizaos start --log-json | jq '.'
```

---

## 10. CHARACTER INTERFACE

### Core Properties
| Property | Type | Required | Purpose |
|----------|------|----------|---------|
| `name` | string | Yes | Display name |
| `bio` | string \| string[] | Yes | Background/personality |
| `id` | UUID | No | Auto-generated |
| `username` | string | No | Social platform ID |
| `system` | string | No | Custom system prompt override |
| `templates` | object | No | Custom prompt templates |
| `adjectives` | string[] | No | Character traits |
| `topics` | string[] | No | Knowledge domains |
| `knowledge` | array | No | Facts, files, directories |
| `messageExamples` | array[][] | No | 2D conversation samples |
| `postExamples` | string[] | No | Social media examples |
| `style` | object | No | Writing rules by context |
| `plugins` | string[] | No | Plugin package names |
| `settings` | object | No | Configuration params |
| `secrets` | object | No | Sensitive credentials |

### Message Examples Format
```typescript
messageExamples: [
  [
    { name: "{{user}}", content: { text: "..." } },
    { name: "AgentName", content: { text: "..." } }
  ]
]
```

### Style Object
```typescript
style: {
  all: [],    // Universal rules
  chat: [],   // Conversation-specific
  post: []    // Social media conventions
  // Custom keys for plugin-specific styles
}
```

### Knowledge Array
- String facts: `"I was founded in 2024"`
- File references: `{ path: "docs/guide.md", shared: boolean }`
- Directory objects: `{ directory: "docs/", shared: boolean }`

### Conditional Plugin Loading
Plugins can be loaded based on environment variables for platform-specific integrations.

---

## 11. MEMORY AND STATE SYSTEM

### Core Memory Interface
```typescript
Memory {
  id: UUID
  entityId: UUID      // creator
  agentId: UUID
  roomId: UUID
  worldId?: UUID
  content: Content    // text, actions, reply refs, metadata
  embedding?: number[] // vector representation
  // Timestamps, dedup flags, similarity scores
}

Content {
  text: string
  actions?: string[]
  replyRef?: Reference
  metadata?: Record<string, any>
}
```

### Memory Lifecycle
1. **Creation**: `createMemory(memory, tableName, unique?)` with optional deduplication
2. **Storage**: Via `IDatabaseAdapter` with text, embeddings, metadata, relationships
3. **Retrieval**: `getMemories()` by room/entity/time; `searchMemories()` for semantic/keyword

### Context Management
- Default: 32 messages, 4000 token limit with pruning
- **Recency-based**: Most recent via `getMemories()` descending timestamp
- **Importance-based**: High-scoring filtered by metadata threshold
- **Hybrid**: Recent (10) + important (5), deduplicated

### State Composition
```typescript
State {
  values: { [key]: unknown }       // KV store from providers
  data: StateData                  // Structured cache
  text: string                     // Formatted context
}

StateData {
  room, world, entity              // Cached entity data
  providers: Record<string, Record> // Provider results
  actionPlan, actionResults        // Action tracking
}
```

### Memory Types
- **Short-term**: Working buffer (max 50, FIFO)
- **Long-term**: Metadata type, importance score, access counts; preserved during pruning
- **Knowledge**: Static character + dynamic facts with confidence metadata

### Advanced Operations
- **Semantic Search**: Vector similarity, threshold 0.75 default, ANN indexing (Annoy)
- **Hybrid Search**: Semantic + keyword extraction + text search, deduplicated/ranked
- **Batch**: `embedBatch()`, `batchCreateMemories()` in single DB transactions

### Caching
- L1: In-memory, 100-item capacity
- L2: LRU, 1000 items, 5-minute TTL, promote on hit
- DB: B-tree indexes on room/entity/timestamp; IVF index for embeddings

### Memory Networks
- Directed graphs: causes, effects, related, references
- Traversal depth-limited to 3 levels

### Temporal Patterns
- Exponential decay: 7-day half-life, modified by importance
- Temporal windows: +/- 30 minutes default

### Multi-Agent Memory
- Visibility: public/private/selective
- Per-agent permissions: read/write/delete
- Sync via last-sync timestamps

### Pruning
- Time-based: >30 days (excludes long-term)
- Importance-based: top-scored when >10,000 limit

---

## 12. RUNTIME AND LIFECYCLE

### AgentRuntime Components
- `agentId`: UUID
- `character`: Personality config
- `adapter`: Database handler
- `services`: Background processes
- `models`: LLM handlers by type

### Initialization Sequence (4 phases)
1. **Configuration**: Character validation, defaults
2. **Initialization**: DB connection, plugin resolution
3. **Runtime**: Message processing loop, service management
4. **Shutdown**: Graceful cleanup, resource release

### Plugin Loading
- Dependency graph construction
- Circular dependency detection
- Topological sorting
- Ordered init: services -> actions -> providers -> evaluators -> models

### Lifecycle Hooks
- `init()`: Plugin loading
- `start()` / `stop()`: Service lifecycle
- `beforeMessage()` / `afterMessage()`: Message interception
- `beforeAction()` / `afterAction()`: Action hooks

### Service Management
- Start in dependency order
- Stop in reverse order
- Failures: fatal or graceful (configurable)
- Discovery: `runtime.getService<T>(name)`

### Multi-Agent Systems
- **MultiAgentCoordinator**: Message routing, offline queues, broadcast
- **Parent-Child**: Setting/plugin inheritance, shared DB adapters, separate runtimes

### Production Patterns
- Lazy loading: plugins on-demand
- Eager loading: pre-warm caches, compile templates
- Circuit breakers: prevent cascade failures
- Exponential backoff for retries
- Monitoring: message count, response latency, error rates, memory usage

### Error Recovery
- Configurable retry limits
- Circuit breaker patterns
- Fallback response generation
- Critical error admin notifications

### Deployment Models
- **Single Agent**: Express HTTP with `/message` and `/health`
- **Agent Swarm**: Shared DB adapter, event-based inter-agent comms
- **Edge**: SQLite, reduced context, offline caching, sync-when-online

---

## 13. PERSONALITY AND BEHAVIOR

### Bio Configuration
```typescript
bio: string | string[]  // Arrays for complex, strings for simple
knowledge: string[]     // Facts and experience
```

### Behavioral Traits
```typescript
adjectives: ["helpful", "patient", "knowledgeable"]  // Harmonious, no contradictions
topics: ["JavaScript", "TypeScript", "React"]         // Knowledge boundaries
```

### Message Examples (Most Powerful Tool)
```typescript
messageExamples: [
  [
    { name: "{{user}}", content: { text: "user input" }},
    { name: "Assistant", content: { text: "agent response" }}
  ]
]
```
Cover: greetings, problem-solving, knowledge boundaries, teaching, error handling.

### Style System (3 contexts)
```typescript
style: {
  all: [],    // "Use active voice", "Include examples", "Admit uncertainty"
  chat: [],   // Markdown, digestible chunks, clarifying questions
  post: []    // Hook readers, platform-appropriate length, CTAs
}
```

### Response Templates
```typescript
templates: {
  greeting: ({ timeOfDay }) => { /* contextual */ },
  errorHelp: ({ errorType, context }) => { /* debugging */ },
  success: ({ achievement }) => { /* celebration */ }
}
```

### Personality Archetypes
1. **The Helper**: Patient, thorough, friendly
2. **The Expert**: Deep knowledge, analytical, precise
3. **The Companion**: Empathetic, encouraging, warm
4. **The Analyst**: Data-driven, objective, methodical

### Knowledge Integration
```typescript
plugins: ['@elizaos/plugin-openai', '@elizaos/plugin-knowledge']
```
```
LOAD_DOCS_ON_STARTUP=true
CTX_KNOWLEDGE_ENABLED=true
MAX_INPUT_TOKENS=4000
MAX_OUTPUT_TOKENS=2000
```

### Advanced Features
- Multi-persona switching via templates
- Personality evolution via knowledge array tracking
- Contextual shifts based on channel type, time, situation

---

## 14. PROJECTS

### Structure (Single Agent)
```
my-project/
  src/
    index.ts          # Entry point
    character.ts      # Agent personality
    __tests__/
    frontend/         # Web UI (optional)
    plugins/          # Custom plugins
  .env
  package.json
  tsconfig.json
  dist/               # Compiled
  node_modules/
  .eliza/             # Runtime data
  scripts/
```

### Structure (Multi-Agent)
```
multi-agent-project/
  src/
    index.ts          # Orchestration
    agents/
      eliza.ts, spartan.ts, billy.ts, bobby.ts
    plugins/          # Shared
    __tests__/
  .env                # Shared
```

### Single Agent Entry Point
```typescript
import { Project, ProjectAgent, IAgentRuntime } from '@elizaos/core';
import { character } from './character';

export const projectAgent: ProjectAgent = {
  character,
  init: async (runtime: IAgentRuntime) => {
    console.log(`Initializing ${character.name}`);
  },
  // plugins: [customPlugin]
};

const project: Project = { agents: [projectAgent] };
export default project;
```

### Multi-Agent Orchestration
```typescript
import { Project, ProjectAgent } from '@elizaos/core';
import { eliza, spartan, billy, bobby } from './agents';

const elizaAgent: ProjectAgent = {
  character: eliza,
  init: async (runtime) => { await setupElizaKnowledge(runtime); }
};

const project: Project = {
  agents: [elizaAgent, spartanAgent, billyAgent, bobbyAgent]
};
export default project;
```

### Key Concepts
- Project HAS one or many agents
- Agents IMPORT/USE plugins via character config
- Plugins are REUSABLE across projects and agents
- Per-agent model provider: `settings: { modelProvider: 'anthropic' }`

### Multi-Agent Communication
- Shared memory systems
- Message passing via runtime
- Event-driven coordination
- Shared service instances

### CLI Workflows
```bash
elizaos create --type project   # Initialize
elizaos start                   # Run
elizaos dev                     # Hot-reload
elizaos start --character ./custom.ts
```

### Reset
```bash
rm -rf node_modules dist .eliza && bun install
```

---

## 15. ENVIRONMENT VARIABLES

### Server Security
```
ELIZA_SERVER_AUTH_TOKEN      # API auth (header: X-API-KEY)
                             # Unset = no auth; recommended for production
```

### Web UI
```
ELIZA_UI_ENABLE             # true/false; default: enabled in dev, disabled in prod
```

### Environment Mode
```
NODE_ENV                    # development (default) or production
                            # Affects CSP, error verbosity, debugging features
```

### Database
```
POSTGRES_URL                # PostgreSQL connection string (triggers PG adapter)
PGLITE_DATA_DIR             # PGLite directory (default: ./.eliza/.elizadb)
```

### Model Providers
```
OPENAI_API_KEY
ANTHROPIC_API_KEY
OPENROUTER_API_KEY
OPENROUTER_EMBEDDING_MODEL
```

### Platform Credentials (per-platform sections above)
```
# Discord
DISCORD_APPLICATION_ID
DISCORD_API_TOKEN

# Telegram
TELEGRAM_BOT_TOKEN

# Twitter (OAuth 1.0a!)
TWITTER_API_KEY
TWITTER_API_SECRET_KEY
TWITTER_ACCESS_TOKEN
TWITTER_ACCESS_TOKEN_SECRET

# Farcaster
FARCASTER_NEYNAR_API_KEY
FARCASTER_SIGNER_UUID
FARCASTER_FID
```

### Knowledge
```
LOAD_DOCS_ON_STARTUP=true
KNOWLEDGE_PATH
CTX_KNOWLEDGE_ENABLED=true
MAX_INPUT_TOKENS=4000
MAX_OUTPUT_TOKENS=2000
```

### Runtime
```
LOG_LEVEL                    # debug, trace, error, info
CONVERSATION_LENGTH=20
RESPONSE_TIMEOUT=3600000
```

---

## CROSS-CUTTING PATTERNS AND BEST PRACTICES

### Universal Plugin Integration Pattern
```
Import plugin -> Create AgentRuntime with plugin -> Configure env vars -> Start runtime
```

### Error Handling Strategy (All Platforms)
1. Rate limits (429): Exponential backoff with reset timers
2. Auth errors (401/403): Log and don't retry
3. Bad requests (400): Log validation error, don't retry
4. Network errors: Retry with exponential backoff (base ~1000ms, 2^attempt)
5. Circuit breakers for cascading failure prevention

### Caching Strategy (All Platforms)
- LRU caches with configurable TTL
- Multi-level: in-memory L1 + LRU L2
- TTL validation with automatic refresh
- Cache-first reads with fallback to API/DB

### State Management Pattern
- Entities map to users across all platforms
- Rooms map to channels/chats/conversations
- Worlds map to servers/groups/namespaces
- Memory persists cross-platform with unified schema

### Message Processing Pattern (Universal)
```
Platform Event -> Platform Plugin (format conversion) -> Bootstrap Plugin (processing) -> LLM (generation) -> Bootstrap (action/eval) -> Platform Plugin (delivery)
```

### Template Override Priority
1. Character-level templates (highest)
2. Plugin-level templates
3. Bootstrap defaults (lowest)

### Security Best Practices
- Store ALL credentials in environment variables
- Never hardcode tokens
- Separate dev/production API keys
- Validate inputs before processing
- Implement rate limiting at all API boundaries
- Enable debug logging only in development
