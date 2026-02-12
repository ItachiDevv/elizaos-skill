# ElizaOS Plugin Documentation - Complete Extraction

## Table of Contents
1. [Knowledge Plugin Overview](#1-knowledge-plugin-overview)
2. [Knowledge Plugin Complete Documentation](#2-knowledge-plugin-complete-documentation)
3. [Contextual Embeddings](#3-contextual-embeddings)
4. [Architecture & Flow](#4-architecture--flow)
5. [Knowledge Plugin Examples](#5-knowledge-plugin-examples)
6. [Knowledge Plugin Quick Start](#6-knowledge-plugin-quick-start)
7. [Discord Plugin](#7-discord-plugin)
8. [Discord Developer Guide](#8-discord-developer-guide)
9. [Twitter Plugin](#9-twitter-plugin)
10. [Farcaster Plugin](#10-farcaster-plugin)

---

## 1. Knowledge Plugin Overview

**Source:** https://docs.elizaos.ai/plugin-registry/knowledge

### Package
`@elizaos/plugin-knowledge` - RAG system for intelligent document management and retrieval.

### Key Features
- **Zero Configuration** - sensible defaults
- **Multiple Formats** - PDF, TXT, MD, DOCX, CSV, and more
- **Intelligent Processing** - smart chunking and contextual embeddings
- **Cost Optimization** - 90% cost reduction with caching

### Core Capabilities

**Document Processing:**
- Automatic text extraction from PDFs, Word documents
- Smart chunking with configurable overlap
- Content-based deduplication
- Metadata preservation and enrichment

**Retrieval & RAG:**
- Semantic search with vector embeddings
- Automatic context injection into conversations
- Relevance scoring and ranking
- Multi-modal retrieval support

**Management Interface:**
- Web-based document browser at http://localhost:3000
- Upload, view, and delete documents
- Search and filter capabilities
- Real-time processing status

### Installation

```bash
elizaos plugins add @elizaos/plugin-knowledge
```

```bash
bun add @elizaos/plugin-knowledge
```

### Supported File Types
- Documents: PDF, DOCX, TXT, MD
- Data: CSV, JSON, XML
- Web: URLs, HTML
- Code: JS, TS, PY, JAVA, CPP, HTML, CSS

### Advanced Features
- Contextual Embeddings: 50% better retrieval accuracy
- Architecture & Flow documentation
- REST API endpoints and TypeScript interfaces

---

## 2. Knowledge Plugin Complete Documentation

**Source:** https://docs.elizaos.ai/plugin-registry/knowledge/complete-documentation

### Architecture Components

#### KnowledgeService Class

```typescript
class KnowledgeService extends Service {
  static readonly serviceType = 'knowledge';
  private knowledgeConfig: KnowledgeConfig;
  private knowledgeProcessingSemaphore: Semaphore;

  constructor(runtime: IAgentRuntime, config?: Partial<KnowledgeConfig>) {
    super(runtime);
    this.knowledgeProcessingSemaphore = new Semaphore(10);

    this.knowledgeConfig = {
      CTX_KNOWLEDGE_ENABLED: parseBooleanEnv(config?.CTX_KNOWLEDGE_ENABLED),
      LOAD_DOCS_ON_STARTUP: loadDocsOnStartup,
      MAX_INPUT_TOKENS: config?.MAX_INPUT_TOKENS,
      MAX_OUTPUT_TOKENS: config?.MAX_OUTPUT_TOKENS,
      EMBEDDING_PROVIDER: config?.EMBEDDING_PROVIDER,
      TEXT_PROVIDER: config?.TEXT_PROVIDER,
      TEXT_EMBEDDING_MODEL: config?.TEXT_EMBEDDING_MODEL,
    };

    if (this.knowledgeConfig.LOAD_DOCS_ON_STARTUP) {
      this.loadInitialDocuments();
    }
  }

  async addKnowledge(options: AddKnowledgeOptions): Promise<{
    clientDocumentId: string;
    storedDocumentMemoryId: UUID;
    fragmentCount: number;
  }> {
    const contentBasedId = generateContentBasedId(options.content, agentId, {
      includeFilename: options.originalFilename,
      contentType: options.contentType,
      maxChars: 2000,
    });

    const existingDocument = await this.runtime.getMemoryById(contentBasedId);
    if (existingDocument) {
      return { clientDocumentId: contentBasedId, ... };
    }

    return this.processDocument({ ...options, clientDocumentId: contentBasedId });
  }

  async getKnowledge(
    message: Memory,
    scope?: { roomId?: UUID; worldId?: UUID; entityId?: UUID }
  ): Promise<KnowledgeItem[]> {
    const embedding = await this.runtime.useModel(ModelType.TEXT_EMBEDDING, {
      text: message.content.text,
    });

    const fragments = await this.runtime.searchMemories({
      tableName: 'knowledge',
      embedding,
      query: message.content.text,
      ...scope,
      count: 20,
      match_threshold: 0.1,
    });

    return fragments.map(fragment => ({
      id: fragment.id,
      content: fragment.content,
      similarity: fragment.similarity,
      metadata: fragment.metadata,
    }));
  }

  async enrichConversationMemoryWithRAG(
    memoryId: UUID,
    ragMetadata: {
      retrievedFragments: Array<{
        fragmentId: UUID;
        documentTitle: string;
        similarityScore?: number;
        contentPreview: string;
      }>;
      queryText: string;
      totalFragments: number;
      retrievalTimestamp: number;
    }
  ): Promise<void> {
    // Enriches conversation memories with RAG usage data
  }
}
```

#### Document Processing Pipeline

```typescript
private async processDocument(options: AddKnowledgeOptions): Promise<{
  clientDocumentId: string;
  storedDocumentMemoryId: UUID;
  fragmentCount: number;
}> {
  let fileBuffer: Buffer | null = null;
  let extractedText: string;
  let documentContentToStore: string;
  const isPdfFile = contentType === 'application/pdf';

  if (isPdfFile) {
    fileBuffer = Buffer.from(content, 'base64');
    extractedText = await extractTextFromDocument(fileBuffer, contentType, originalFilename);
    documentContentToStore = content; // store base64 for PDFs
  } else if (isBinaryContentType(contentType, originalFilename)) {
    fileBuffer = Buffer.from(content, 'base64');
    extractedText = await extractTextFromDocument(fileBuffer, contentType, originalFilename);
    documentContentToStore = extractedText;
  } else {
    if (looksLikeBase64(content)) {
      const decodedBuffer = Buffer.from(content, 'base64');
      extractedText = decodedBuffer.toString('utf8');
      documentContentToStore = extractedText;
    } else {
      extractedText = content;
      documentContentToStore = content;
    }
  }

  const documentMemory = createDocumentMemory({
    text: documentContentToStore,
    agentId,
    clientDocumentId,
    originalFilename,
    contentType,
    worldId,
    fileSize: fileBuffer ? fileBuffer.length : extractedText.length,
    documentId: clientDocumentId,
    customMetadata: metadata,
  });

  await this.runtime.createMemory(documentMemory, 'documents');

  const fragmentCount = await processFragmentsSynchronously({
    runtime: this.runtime,
    documentId: clientDocumentId,
    fullDocumentText: extractedText,
    agentId,
    contentType,
    roomId: roomId || agentId,
    entityId: entityId || agentId,
    worldId: worldId || agentId,
    documentTitle: originalFilename,
  });

  return { clientDocumentId, storedDocumentMemoryId, fragmentCount };
}
```

#### Supported File Format Extractors

```typescript
const supportedFormats = {
  'application/pdf': extractPDF,
  'application/vnd.openxmlformats-officedocument.wordprocessingml.document': extractDOCX,
  'text/plain': (buffer) => buffer.toString('utf-8'),
  'text/markdown': (buffer) => buffer.toString('utf-8'),
  'application/json': (buffer) => JSON.stringify(JSON.parse(buffer.toString('utf-8')), null, 2)
};
```

#### Content-Based Deduplication

```typescript
const contentBasedId = generateContentBasedId(content, agentId, {
  includeFilename: options.originalFilename,
  contentType: options.contentType,
  maxChars: 2000
});

const existingDocument = await this.runtime.getMemoryById(contentBasedId);
if (existingDocument) {
  return { clientDocumentId: contentBasedId, ... };
}
```

### Chunking Configuration

```typescript
const defaultChunkOptions = {
  chunkSize: 500,        // tokens
  overlapSize: 100,      // tokens
  separators: ['\n\n', '\n', '. ', ' '],
  keepSeparator: true
};
```

### Knowledge Ingestion Methods

```typescript
// File upload (API endpoint sends base64-encoded content)
const result = await knowledgeService.addKnowledge({
  content: base64EncodedContent,
  originalFilename: 'document.pdf',
  contentType: 'application/pdf',
  agentId: agentId,
  metadata: {
    tags: ['documentation', 'manual']
  }
});

// Direct text addition (internal use)
await knowledgeService._internalAddKnowledge({
  id: generateContentBasedId(content, agentId),
  content: { text: "Important information..." },
  metadata: {
    type: MemoryType.DOCUMENT,
    source: 'direct'
  }
});

// Character knowledge (loaded automatically from character definition)
await knowledgeService.processCharacterKnowledge([
  "Path: knowledge/facts.md\nKey facts about the product...",
  "Another piece of character knowledge..."
]);
```

### Embedding Generation

```typescript
async function generateEmbeddings(chunks: string[]): Promise<number[][]> {
  const embeddings = await embedder.embedMany(chunks);
  return embeddings;
}

const batchSize = 10;
for (let i = 0; i < chunks.length; i += batchSize) {
  const batch = chunks.slice(i, i + batchSize);
  const embeddings = await generateEmbeddings(batch);
  await storeEmbeddings(embeddings);
  await sleep(1000); // 1-second delay between batches
}
```

### Semantic Search Implementation

```typescript
async function searchKnowledge(query: string, limit: number = 10): Promise<KnowledgeItem[]> {
  const queryEmbedding = await embedder.embed(query);

  const results = await vectorStore.searchMemories({
    tableName: "knowledge_embeddings",
    agentId: runtime.agentId,
    embedding: queryEmbedding,
    match_threshold: 0.7,
    match_count: limit,
    unique: true
  });

  return results.map(result => ({
    id: result.id,
    content: result.content.text,
    score: result.similarity,
    metadata: result.metadata
  }));
}
```

### Vector Storage Structure

```typescript
// Document storage
{
  id: "doc_123",
  content: "Full document text",
  metadata: {
    source: "upload",
    filename: "report.pdf",
    createdAt: "2024-01-20T10:00:00Z",
    hash: "sha256_hash"
  }
}

// Vector storage
{
  id: "vec_456",
  documentId: "doc_123",
  chunkIndex: 0,
  embedding: [0.123, -0.456, ...],
  content: "Chunk text",
  metadata: {
    position: { start: 0, end: 500 }
  }
}
```

### Core Actions

#### PROCESS_KNOWLEDGE
- Accepts file paths, direct text, or multiple file types
- Automatically fragments content for search
- Returns processing status

#### SEARCH_KNOWLEDGE
- Triggered by explicit "Search your knowledge" requests
- Returns top 3 results with formatted snippets

### Knowledge Provider
- Runs on EVERY message to find relevant context
- Retrieves up to 5 most relevant fragments
- Caps knowledge at approximately 4000 tokens
- Enriches conversation memories with knowledge usage metadata

### REST API Endpoints

**POST /knowledge/upload**
- Accepts: multipart/form-data with file and metadata
- Returns: document ID and processing status

**GET /knowledge/documents**
- Parameters: page, limit
- Returns: document list with metadata and chunk counts

**DELETE /knowledge/documents/{id}**
- Removes document and associated embeddings

**GET /knowledge/search**
- Parameters: q (query), limit
- Returns: ranked results with similarity scores

### TypeScript Interfaces

```typescript
interface AddKnowledgeOptions {
  agentId?: UUID;
  worldId: UUID;
  roomId: UUID;
  entityId: UUID;
  clientDocumentId: UUID;
  contentType: string;
  originalFilename: string;
  content: string;
  metadata?: Record<string, unknown>;
}

interface KnowledgeConfig {
  CTX_KNOWLEDGE_ENABLED: boolean;
  LOAD_DOCS_ON_STARTUP: boolean;
  MAX_INPUT_TOKENS?: string | number;
  MAX_OUTPUT_TOKENS?: string | number;
  EMBEDDING_PROVIDER?: string;
  TEXT_PROVIDER?: string;
  TEXT_EMBEDDING_MODEL?: string;
}

interface TextGenerationOptions {
  provider?: 'anthropic' | 'openai' | 'openrouter' | 'google';
  modelName?: string;
  maxTokens?: number;
  cacheDocument?: string;
  cacheOptions?: { type: 'ephemeral' };
  autoCacheContextualRetrieval?: boolean;
}

interface KnowledgeItem {
  id: UUID;
  content: string;
  similarity: number;
  metadata: Record<string, unknown>;
}
```

### Rate Limiting

```typescript
const rateLimiter = {
  maxConcurrent: 5,
  requestsPerMinute: 60,
  tokensPerMinute: 40000
};
```

### Retry Logic with Exponential Backoff

```typescript
async function withRetry(fn, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await sleep(Math.pow(2, i) * 1000);
    }
  }
}
```

### Memory/Cache Management

```typescript
setInterval(() => {
  knowledgeService.clearCache();
}, 3600000); // Every hour

async function processLargeFile(path: string) {
  const stream = createReadStream(path);
  const chunks = [];
  for await (const chunk of stream) {
    chunks.push(await processChunk(chunk));
  }
  return chunks;
}
```

### Service Usage in Runtime

```typescript
const knowledgeService = runtime.getService<KnowledgeService>('knowledge');

const result = await knowledgeService.addKnowledge({
  content: documentContent,
  originalFilename: 'guide.pdf',
  contentType: 'application/pdf',
  worldId: runtime.agentId,
  roomId: message.roomId,
  entityId: message.entityId
});

const results = await knowledgeService.getKnowledge(message, {
  roomId: message.roomId,
  worldId: runtime.agentId
});
```

### Configuration Environment Variables

```
LOAD_DOCS_ON_STARTUP=true
CTX_KNOWLEDGE_ENABLED=true
EMBEDDING_PROVIDER=openai
TEXT_EMBEDDING_MODEL=text-embedding-3-small
LOG_LEVEL=debug
```

### Character Configuration

```json
{
  "name": "SupportAgent",
  "plugins": ["@elizaos/plugin-knowledge"],
  "knowledge": [
    "Default knowledge statement 1",
    "Default knowledge statement 2"
  ]
}
```

### Best Practices
- Use descriptive filenames, logical folder structures, and metadata tags
- Enable caching for repeated queries
- Start with chunk size of 500 tokens
- Implement rate limiting with batch processing
- Periodically clear cache for large knowledge bases
- Validate file uploads, sanitize extracted text
- Implement role-based access controls
- Avoid storing secrets (passwords, API keys)

### Troubleshooting
- **Document Loading Issues**: Verify file permissions with `ls -la agent/docs/`
- **Poor Retrieval**: Adjust `EMBEDDING_CHUNK_SIZE=800` and `EMBEDDING_OVERLAP_SIZE=200`
- **Rate Limiting**: Implement exponential backoff retry logic

---

## 3. Contextual Embeddings

**Source:** https://docs.elizaos.ai/plugin-registry/knowledge/contextual-embeddings

### Core Concept
Contextual embeddings improve retrieval accuracy by enriching text chunks with surrounding context before generating embeddings. Based on Anthropic's contextual retrieval techniques. Claims up to 50% improvement in retrieval accuracy.

### Traditional vs. Contextual Approach
- **Traditional**: Embeds isolated text chunks, losing important surrounding information
- **Contextual**: An LLM adds relevant context to each chunk before embedding

### How It Works (4-step process)
1. **Document Analysis** - Full document passed to LLM with each chunk
2. **Context Generation** - LLM identifies relevant surrounding context
3. **Chunk Enrichment** - Original chunk preserved with added context
4. **Embedding** - Enriched chunk embedded using configured model

### Configuration

```env
CTX_KNOWLEDGE_ENABLED=true
TEXT_PROVIDER=openrouter|openai|anthropic|google
TEXT_MODEL=anthropic/claude-3-haiku
OPENROUTER_API_KEY=your-key
OPENROUTER_EMBEDDING_MODEL=openai/text-embedding-3-large
OLLAMA_API_ENDPOINT=http://localhost:11434/api  # for fallback
LOG_LEVEL=debug  # for monitoring
```

### Technical Specifications

| Parameter | Value |
|-----------|-------|
| Chunk Size | 500 tokens (~1,750 characters) |
| Chunk Overlap | 100 tokens |
| Context Target | 60-200 tokens of added context |
| Processing Time (initial) | 1-3 seconds per chunk |
| Processing Time (cached) | 0.1-0.3 seconds per chunk |
| Concurrent Batch Size | up to 30 chunks |
| OpenRouter Cache Duration | 5 minutes |
| Cost Reduction with Caching | 90% |

### Content-Aware Processing
Plugin detects and handles with specialized prompts:
- General text
- PDFs
- Mathematical content
- Code files
- Technical documentation

### OpenRouter Caching Benefits
- First chunk: Caches full document
- Subsequent chunks: Reuses cache (90% cost reduction)
- Cache duration: 5 minutes automatic

### Cost Estimation

| Size | Pages | Chunks | Without Cache | With Cache |
|------|-------|--------|---------------|------------|
| Small | 10 | ~20 | $0.02 | $0.002 |
| Medium | 50 | ~100 | $0.10 | $0.01 |
| Large | 200 | ~400 | $0.40 | $0.04 |

(Based on Claude 3 Haiku pricing)

### Plugin Configuration Examples

**OpenRouter Only:**
```typescript
export const character = {
  name: 'MyAgent',
  plugins: [
    '@elizaos/plugin-openrouter',
    '@elizaos/plugin-knowledge',
  ],
};
```

**OpenRouter + Local Fallback:**
```typescript
export const character = {
  name: 'MyAgent',
  plugins: [
    '@elizaos/plugin-openrouter',
    '@elizaos/plugin-ollama',
    '@elizaos/plugin-knowledge',
  ],
};
```

**Alternative Provider Configurations:**
- OpenAI: `TEXT_PROVIDER=openai`, `TEXT_MODEL=gpt-4o-mini`
- Anthropic + OpenAI: `TEXT_PROVIDER=anthropic`, `TEXT_MODEL=claude-3-haiku-20240307`
- Google Only: `TEXT_PROVIDER=google`, `TEXT_MODEL=gemini-1.5-flash`
- OpenRouter + Google: Requires `@elizaos/plugin-google` for embeddings

### Embedding Models Available
- `text-embedding-3-large` (OpenAI)
- `qwen3-embedding`
- `gemini-embedding`
- `mistral-embed`

### Critical Warnings
- **IMPORTANT**: Embeddings always use the model configured in `useModel(TEXT_EMBEDDING)` from your agent setup. Do NOT mix different embedding models - all documents must use the same embedding model for consistency.
- Configuration values are case-sensitive (`CTX_KNOWLEDGE_ENABLED=true`, not `TRUE`).

### Example Impact

**Without contextual embeddings:**
Query "How do I configure the timeout?" retrieves generic chunk: "Set the timeout to 30 seconds" (ambiguous - which timeout?)

**With contextual embeddings:**
Same query retrieves chunk clarifying: "Redis configuration section, caching layer, session data timeout" (precise context)

### Debug Indicators
- Look for "CTX enrichment ENABLED" or "CTX enrichment DISABLED" in logs
- Enable `LOG_LEVEL=debug` to view enrichment progress, cache metrics, processing times, token usage

### Model Recommendations
- **Claude 3 Haiku**: Best quality/cost balance
- **Gemini 1.5 Flash**: Best speed
- **GPT-4o-mini**: Quality at moderate cost

---

## 4. Architecture & Flow

**Source:** https://docs.elizaos.ai/plugin-registry/knowledge/architecture-flow

### Main Components
- Knowledge Service (orchestration)
- Document Processor (text handling)
- Embedding Provider (vectorization)
- Vector Store (similarity search)
- Document Store (content storage)
- Web Interface (user access)
- Agent Memory (integration)
- Action Processor (request routing)

### Document Processing Pipeline
1. Input detection (files, URLs, direct text)
2. Content extraction and cleaning
3. Hash generation for deduplication
4. Text chunking (sliding window with overlap)
5. Optional contextual enrichment
6. Embedding generation
7. Vector storage with metadata

### Chunking Strategy
Sliding window approach with boundary adjustments:
- Window 1: tokens 0-500
- Window 2: tokens 400-900
- Window 3: tokens 800-1300
- Adjusted to natural boundaries (sentences, paragraphs)

### Retrieval Process
1. Query embedding generation
2. Vector similarity searching
3. Optional metadata filtering
4. Relevance ranking
5. Threshold validation (score > 0.7)
6. Result limiting
7. Context injection into agent responses

### Rate Limiting Parameters
- Token bucket: 150k tokens/min
- Request bucket: 60 requests/min
- Concurrent operation limit: 30 operations

### Caching Configuration
- LRU eviction policy
- TTL: 24 hours
- Max size: 10k entries
- 90% cost reduction with OpenRouter + Claude or Gemini

### Error Handling
- Exponential backoff for rate limits
- 3x retries for network errors
- Chunk size reduction for memory issues

### Storage Distribution
- Document text: 40%
- Vector embeddings: 35%
- Metadata: 15%
- Indexes: 10%

### Performance Characteristics

| Document Size | Processing Time |
|---------------|-----------------|
| Small (<1MB) | ~6 seconds |
| Medium (1-10MB) | ~17 seconds |
| Large (10-50MB) | ~50 seconds |

### Scaling Architecture
Horizontal scaling via load balancer with:
- Shared PostgreSQL + pgvector (Vector Store)
- PostgreSQL (Document Store)
- Redis cache

---

## 5. Knowledge Plugin Examples

**Source:** https://docs.elizaos.ai/plugin-registry/knowledge/examples

### Three Core Loading Methods
1. **Auto-load (Recommended)**: Documents automatically process from `docs/` folder on startup
2. **Web Upload**: Dynamic content via browser at `http://localhost:3000`
3. **Hardcoding**: Only for tiny bits of info (character `knowledge` array)

### Support Bot Character

```typescript
import { type Character } from '@elizaos/core';

export const supportBot: Character = {
  name: 'SupportBot',
  plugins: [
    '@elizaos/plugin-openai',
    '@elizaos/plugin-knowledge',
  ],
  system: 'You are a friendly customer support agent. Answer questions using the support documentation you have learned. Always search your knowledge base before responding.',
  bio: [
    'Expert in product features and troubleshooting',
    'Answers based on official documentation',
    'Always polite and helpful',
  ],
};
```

**Folder Structure:**
```
your-project/
├── docs/
│   ├── product-manual.pdf
│   ├── troubleshooting-guide.md
│   ├── faq.txt
│   └── policies/
│       ├── refund-policy.pdf
│       └── terms-of-service.md
├── .env
└── src/
    └── character.ts
```

### API Documentation Assistant

```typescript
export const apiAssistant: Character = {
  name: 'APIHelper',
  plugins: [
    '@elizaos/plugin-openai',
    '@elizaos/plugin-knowledge',
  ],
  system: 'You are a technical documentation assistant. Help developers by searching your knowledge base for API documentation, code examples, and best practices.',
  topics: [
    'API endpoints and methods',
    'Authentication and security',
    'Code examples and best practices',
    'Error handling and debugging',
  ],
};
```

**Document Structure:**
```
docs/
├── api-reference/
│   ├── authentication.md
│   ├── endpoints.json
│   └── error-codes.csv
├── tutorials/
│   ├── getting-started.md
│   ├── advanced-usage.md
│   └── examples.ts
└── changelog.md
```

### Info Bot (Minimal)

```json
{
  "name": "InfoBot",
  "plugins": [
    "@elizaos/plugin-openai",
    "@elizaos/plugin-knowledge"
  ],
  "knowledge": [
    "Our office is located at 123 Main St",
    "Business hours: 9 AM to 5 PM EST",
    "Contact: support@example.com"
  ],
  "system": "You are a simple information bot. Answer questions using your basic knowledge."
}
```

### Production Setup

**Document Structure:**
```
docs/
├── products/
│   ├── product-overview.pdf
│   ├── pricing-tiers.md
│   └── feature-comparison.xlsx
├── support/
│   ├── installation-guide.pdf
│   ├── troubleshooting.md
│   └── common-issues.txt
├── legal/
│   ├── terms-of-service.pdf
│   ├── privacy-policy.md
│   └── data-processing.txt
└── README.md
```

**Environment:**
```env
OPENAI_API_KEY=sk-...
LOAD_DOCS_ON_STARTUP=true
KNOWLEDGE_PATH=/var/app/support-docs
CTX_KNOWLEDGE_ENABLED=true
OPENROUTER_API_KEY=sk-or-...
```

### Agent Behavior
The knowledge provider automatically injects relevant knowledge into the agent's context. When users ask questions, the system:
1. Searches knowledge base
2. Finds relevant chunks
3. Uses information in responses
No special user commands required.

### Troubleshooting Checklist
- Verify `LOAD_DOCS_ON_STARTUP=true`
- Confirm `docs` folder exists with accessible files
- Check Knowledge tab for loaded document verification
- Try specific question phrasing if searches fail

---

## 6. Knowledge Plugin Quick Start

**Source:** https://docs.elizaos.ai/plugin-registry/knowledge/quick-start

### Minimum Setup
Character configuration requires:
- `@elizaos/plugin-knowledge` in plugins array
- An AI provider plugin (e.g., `@elizaos/plugin-openai`)

### Environment Variables
```env
OPENAI_API_KEY=sk-...             # Required for embeddings
LOAD_DOCS_ON_STARTUP=true         # Auto-load documents
KNOWLEDGE_PATH=/path/to/docs      # Custom folder (optional)
OPENROUTER_API_KEY=sk-or-...      # Alternative provider
OPENROUTER_EMBEDDING_MODEL=openai/text-embedding-3-large
```

### Supported Formats
- **Text files**: .txt, .md, .csv, .json, .xml, .yaml
- **Documents**: .pdf, .doc, .docx
- **Code files**: .js, .ts, .py, .java, .cpp, .html, .css

### Document Loading
Place documents in `docs/` folder at project root. Subfolders organize by category.

### Automatic Actions
- **PROCESS_KNOWLEDGE** - Document ingestion
- **SEARCH_KNOWLEDGE** - Knowledge retrieval

### Web Interface
Access Knowledge Manager at `http://localhost:3000` after `elizaos start`.
Features: upload, search, delete, drag-and-drop.

### OpenRouter Native Embedding Models
- `text-embedding-3-large`
- `qwen3-embedding`
- `gemini-embedding`
- `mistral-embed`

### Troubleshooting
- Verify folder existence and file format support
- Confirm port 3000 availability
- Validate processing through Knowledge tab
- Use simplified search terminology

---

## 7. Discord Plugin

**Source:** https://docs.elizaos.ai/plugin-registry/platform/discord

### Package
`@elizaos/plugin-discord` - Full Discord bot with messages, voice channels, slash commands, and media processing.

### Required Configuration
| Variable | Description |
|----------|-------------|
| `DISCORD_APPLICATION_ID` | Your Discord application ID |
| `DISCORD_API_TOKEN` | Bot authentication token |

### Optional Configuration
| Variable | Description |
|----------|-------------|
| `CHANNEL_IDS` | Restrict bot to specific channels (comma-separated) |
| `DISCORD_VOICE_CHANNEL_ID` | Default voice channel |

---

## 8. Discord Developer Guide

**Source:** https://docs.elizaos.ai/plugin-registry/platform/discord/developer-guide

### Core Architecture (4 Components)

**1. DiscordService** - Primary entry point
- Client initialization with Discord.js intents
- Event registration
- Channel restrictions via `CHANNEL_IDS` parsing
- Component coordination

**2. MessageManager** - Message handling
- Message reception and validation
- Format conversion to elizaOS standards
- Attachment processing (images with vision models, audio/video transcription)
- Response delivery via callbacks

**3. VoiceManager** - Voice channels
- Voice channel connections
- Audio stream capture
- Voice activity detection
- Transcription service integration

**4. Attachment Handler** - Media processing
- Image descriptions
- Audio transcription

### Environment Variables

```env
DISCORD_APPLICATION_ID=<your-app-id>      # Required
DISCORD_API_TOKEN=<your-bot-token>         # Required
CHANNEL_IDS=<comma-separated-ids>          # Optional - restrict channels
DISCORD_VOICE_CHANNEL_ID=<channel-id>      # Optional - default voice
VOICE_ACTIVITY_THRESHOLD=<number>          # Optional - voice detection
DISCORD_TEST_CHANNEL_ID=<channel-id>       # Optional - testing
```

### Required Bot Permissions
**Text Operations:**
- ViewChannel
- SendMessages
- SendMessagesInThreads
- CreatePublicThreads
- CreatePrivateThreads
- EmbedLinks
- AttachFiles
- ReadMessageHistory
- AddReactions
- UseExternalEmojis

**Voice Operations:**
- Connect
- Speak
- UseVAD

**Commands:**
- UseApplicationCommands

### Event Processing Flows

**1. Guild Join:**
- Creates server room
- Emits WORLD_JOINED event
- Registers slash commands

**2. Message Create:**
- Validates permissions
- Routes through MessageManager
- Tracks conversation context

**3. Interaction Create:**
- Routes chat input commands to registered handlers

### Available Actions

**chatWithAttachments**
- Processes messages with media attachments
- Generates responses considering attachment content

**joinVoice**
- Connects bot to specified voice channel

**transcribeMedia**
- Converts audio/video files to text transcripts

### Provider Functions

**channelStateProvider**
- Returns: channel name, type, guild details, member count

**voiceStateProvider**
- Returns: voice channel state, connected members, speaking status

### Multi-Server Support
Each server maintains isolated:
- Conversation contexts
- User relationships
- Channel states
- Voice connections
- Slash commands register per-guild

### Performance Patterns
- **Message caching**: LRU, 1-hour TTL, 1000 max entries
- **Rate limiting**: 30 requests per 60-second window
- **Voice connection pooling**: Connection reuse
- **Error recovery**: Exponential backoff

### Error Handling
| Error | Code | Handling |
|-------|------|----------|
| Unknown Message | 10008 | Graceful skip |
| Missing Access | 50001 | Notify via alternative |
| Missing Permissions | 50013 | Notify via alternative |
| Connection Issues | - | Reconnection scheduling |

---

## 9. Twitter Plugin

**Source:** https://docs.elizaos.ai/plugin-registry/platform/twitter + developer-guide

### Package
`@elizaos/plugin-twitter` - Autonomous posting, timeline monitoring, and intelligent engagement.

### Architecture Components

**Twitter Service** - Manages multiple client instances
- Creates TwitterClientInstance objects
- Starts post, interaction, and timeline services
- Emits WORLD_JOINED events
- Maintains client map storage

**Client Base** - Core operations
- Initializes TwitterApi with OAuth 1.0a credentials
- Verifies credentials and initializes scraper
- Implements `tweet()` with dry-run support
- Manages rate limiting via requestQueue
- Implements cacheManager for tweet storage

**Post Client** - Autonomous posting
- Checks TWITTER_POST_ENABLE
- Handles TWITTER_POST_IMMEDIATELY
- Generates tweet content via LLM
- Validates tweet length (maxTweetLength)
- Creates threads for oversized content
- Schedules posts with configurable intervals + variance

**Interaction Client** - Timeline monitoring
- Monitors when TWITTER_SEARCH_ENABLE is true
- Retrieves home timeline with max_results: 100
- Filters retweets via exclude parameter
- Tracks processed tweets in Set
- Applies weighted or latest algorithm
- Evaluates response logic before replying

**Timeline Client** - Advanced actions
- Checks TWITTER_ENABLE_ACTION_PROCESSING
- Evaluates like, retweet, and quote actions
- Scores actions and executes highest priority

### Authentication
**CRITICAL: Use OAuth 1.0a, NOT OAuth 2.0!**

Setup process:
1. Apply at https://developer.twitter.com
2. Create app in Developer Portal
3. Enable OAuth 1.0a in authentication settings
4. Set permissions to "Read and write"
5. Configure callback URL: `http://localhost:3000/callback`

Token sources:
- apiKey: Consumer API Key
- apiSecretKey: Consumer API Secret
- accessToken: Access Token (requires write permissions)
- accessTokenSecret: Access Token Secret

### Environment Variables - Complete List

**OAuth Credentials (Required):**
```env
TWITTER_API_KEY=<consumer-api-key>
TWITTER_API_SECRET_KEY=<consumer-api-secret>
TWITTER_ACCESS_TOKEN=<access-token>
TWITTER_ACCESS_TOKEN_SECRET=<access-token-secret>
```

**Operational Flags:**
| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| TWITTER_DRY_RUN | boolean | false | Test mode, no actual posts |
| TWITTER_TARGET_USERS | string | - | Comma-separated or "*" |
| TWITTER_RETRY_LIMIT | number | 5 | Max retries |
| TWITTER_POLL_INTERVAL | seconds | 120 | Polling interval |

**Post Settings:**
| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| TWITTER_POST_ENABLE | boolean | - | Enable autonomous posting |
| TWITTER_POST_INTERVAL_MIN | minutes | 90 | Min post interval |
| TWITTER_POST_INTERVAL_MAX | minutes | 180 | Max post interval |
| TWITTER_POST_IMMEDIATELY | boolean | - | Post on startup |
| TWITTER_POST_INTERVAL_VARIANCE | float | 0.2 | Timing variance (0.0-1.0) |

**Interaction Settings:**
| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| TWITTER_SEARCH_ENABLE | boolean | true | Timeline monitoring |
| TWITTER_INTERACTION_INTERVAL_MIN | minutes | 15 | Min interaction interval |
| TWITTER_INTERACTION_INTERVAL_MAX | minutes | 30 | Max interaction interval |
| TWITTER_AUTO_RESPOND_MENTIONS | boolean | true | Auto-respond to mentions |
| TWITTER_AUTO_RESPOND_REPLIES | boolean | true | Auto-respond to replies |
| TWITTER_MAX_INTERACTIONS_PER_RUN | number | 10 | Max interactions per cycle |

**Timeline Algorithm:**
| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| TWITTER_TIMELINE_ALGORITHM | string | - | "weighted" or "latest" |
| TWITTER_TIMELINE_USER_BASED_WEIGHT | number | 3 | User score weight |
| TWITTER_TIMELINE_TIME_BASED_WEIGHT | number | 2 | Time score weight |
| TWITTER_TIMELINE_RELEVANCE_WEIGHT | number | 5 | Relevance score weight |

**Advanced:**
| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| TWITTER_MAX_TWEET_LENGTH | number | 4000 | Max tweet chars |
| TWITTER_DM_ONLY | boolean | false | DM-only mode |
| TWITTER_ENABLE_ACTION_PROCESSING | boolean | - | Enable action processing |
| TWITTER_ACTION_INTERVAL | minutes | 240 | Action processing interval |

### Timeline Algorithms

**Weighted Algorithm** composite scoring:
```
Combined Score = (userScore * userWeight) + (timeScore * timeWeight) + (relevanceScore * relevanceWeight)
```

**User Scoring Factors:**
| Factor | Score |
|--------|-------|
| Target users | 10 (maximum) |
| Base score | 5 |
| Verified accounts | +2 |
| Followers > 10,000 | +1 |
| Following ratio > 0.8 | +1 |
| Previous interactions | +1 |
| Maximum cap | 10 |

**Time Scoring:**
- Decreases 1 point per 2 hours of age
- Maximum: 10 (newest tweets)

**Latest Algorithm:** Simple chronological sort with maxInteractionsPerRun limit.

### Message Processing

**Tweet Generation:**
- Builds context from recent tweets, trending topics, character, postExamples
- Uses LLM with `buildTweetPrompt()`
- Validates with `validateTweet()`
- Enforces maxTokens: 100

**Response Generation:**
- Retrieves conversation thread
- Builds messages array with tweet author context
- Generates contextual response via LLM
- Returns string or null
- Enforces maxTokens: 100

### Rate Limiting

**RequestQueue Class:**
- Queue array of functions
- Processing flag prevents concurrency
- Calls `rateLimiter.waitIfNeeded()` before execution
- 100ms delay between requests
- 429 errors: `pause(retryAfter * 1000)`

**RateLimiter Class:**
- Maintains `windows: Map<endpoint, RateLimitWindow>`
- Tracks `window.count`, `window.limit`, `window.resetTime`
- Resets expired windows
- Calculates waitTime as `resetTime - Date.now()`

### Error Handling

| Code | Meaning | Action |
|------|---------|--------|
| 429 | Rate limit | Pause + retry after reset |
| 401 | Auth failed | Don't retry |
| 403 | Permission denied | Don't retry |
| 400 | Bad request | Don't retry |
| ECONNRESET/ETIMEDOUT | Network | Retry |

**Retry Logic:**
```typescript
// Exponential backoff: baseDelay * Math.pow(2, i)
// Defaults: maxRetries=3, baseDelay=1000ms
```

### Integration Examples

**Basic Setup:**
```typescript
const runtime = new AgentRuntime({
  plugins: [bootstrapPlugin, twitterPlugin],
  character: {
    name: "TwitterBot",
    clients: ["twitter"],
    postExamples: [...],
    settings: {
      TWITTER_API_KEY: process.env.TWITTER_API_KEY,
      TWITTER_API_SECRET_KEY: process.env.TWITTER_API_SECRET_KEY,
      TWITTER_ACCESS_TOKEN: process.env.TWITTER_ACCESS_TOKEN,
      TWITTER_ACCESS_TOKEN_SECRET: process.env.TWITTER_ACCESS_TOKEN_SECRET,
      TWITTER_POST_ENABLE: "true"
    }
  }
});
await runtime.start();
```

**Multi-Account Setup:**
```typescript
const mainAccount = await twitterService.createClient(runtime, 'main-account', config);
const supportAccount = await twitterService.createClient(runtime, 'support-account', config);
mainAccount.post.start();
supportAccount.interaction.start();
```

### Performance Optimization

**LRUCache Configuration:**
| Cache | Max Entries | TTL |
|-------|-------------|-----|
| Tweet cache | 10,000 | 1 hour |
| User cache | 5,000 | 24 hours |
| Timeline cache | - | 1 minute |

**Batch Operations:**
- `client.v2.users(userIds)` for batch user lookup
- `client.v2.tweets(tweetIds)` for batch tweet lookup
- `Promise.all()` for parallel processing

**Memory Management:**
- maxProcessedTweets: 10,000 limit
- Cleanup interval: 1 hour
- GC trigger if available

### Character Configuration
Settings override environment variables:
```typescript
const character = {
  name: "TwitterBot",
  clients: ["twitter"],
  postExamples: ["Example tweet 1", "Example tweet 2"],
  settings: {
    TWITTER_POST_ENABLE: "true",
    TWITTER_POST_INTERVAL_MIN: "60"
  }
};
```

### Best Practices
1. Always use OAuth 1.0a for user context
2. Store credentials securely
3. Regenerate tokens after permission changes
4. Implement proper backoff strategies
5. Cache frequently accessed data
6. Use batch endpoints when possible
7. Provide diverse postExamples
8. Vary posting times with variance parameter
9. Log all errors with context
10. Monitor memory usage

---

## 10. Farcaster Plugin

**Source:** https://docs.elizaos.ai/plugin-registry/platform/farcaster + developer-guide

### Package
`@elizaos/plugin-farcaster` - Engage with Farcaster's decentralized social network through casting, replies, and interactions via Neynar API.

### Installation
```bash
bun add @elizaos/plugin-farcaster
npm install @elizaos/plugin-farcaster
pnpm add @elizaos/plugin-farcaster
```

### Core Features

**Casting Capabilities:**
- Autonomous posting based on agent personality
- Reply chains and threaded conversations
- Image, link, and frame embedding
- Time-based scheduling

**Engagement Features:**
- Mention and reply monitoring
- Like/recast functionality
- Follow/unfollow automation
- Channel-specific posting

**Hub Integration:**
- Direct hub API access
- Cryptographic message signing
- Farcaster protocol v2 compliance

### Environment Variables - Complete List

**Required:**
| Variable | Description |
|----------|-------------|
| FARCASTER_NEYNAR_API_KEY | Authentication credential for Neynar API |
| FARCASTER_SIGNER_UUID | Neynar-generated signer identifier |
| FARCASTER_FID | Your Farcaster ID number |

**Feature Toggles:**
| Variable | Default | Description |
|----------|---------|-------------|
| ENABLE_CAST | true | Autonomous casting enabled |
| ENABLE_ACTION_PROCESSING | false | Process interactions |
| FARCASTER_DRY_RUN | false | Test mode, no actual posts |

**Timing (all in minutes):**
| Variable | Default | Description |
|----------|---------|-------------|
| CAST_INTERVAL_MIN | 90 | Min cast interval |
| CAST_INTERVAL_MAX | 180 | Max cast interval |
| FARCASTER_POLL_INTERVAL | 2 | Mention polling interval |
| ACTION_INTERVAL | 5 | Action processing interval |

**Other:**
| Variable | Default | Description |
|----------|---------|-------------|
| CAST_IMMEDIATELY | false | Cast on startup |
| ACTION_TIMELINE_TYPE | ForYou | Feed strategy (ForYou) |
| MAX_ACTIONS_PROCESSING | 1 | Max actions per cycle |
| MAX_CAST_LENGTH | 320 | Max cast characters |
| FARCASTER_DEBUG | false | Debug logging |

### Actions

**SEND_CAST** - Posts new casts to Farcaster

**REPLY_TO_CAST** - Responds to existing casts

### Providers

**farcasterProfile** - Returns agent profile data (FID, username, bio)

**farcasterTimeline** - Supplies recent timeline casts for context

### Events

**handleCastSent:**
- Triggered on successful casting
- Stores cast hash, thread ID, message ID, platform metadata

**handleMessageReceived:**
- Processes incoming messages
- Creates memories from cast, profile, and thread data

### Manager Classes

**FarcasterAgentManager:**
- Orchestrates all operations
- Components: client, cast manager, interaction manager

**FarcasterCastManager:**
- Handles autonomous periodic posting
- Respects interval settings (CAST_INTERVAL_MIN/MAX)

**FarcasterInteractionManager:**
- Polls for mentions at FARCASTER_POLL_INTERVAL
- Processes up to MAX_ACTIONS_PROCESSING per cycle

### Service Classes

**FarcasterService:**
- Main coordinator extending Service class
- Lifecycle methods: initialize, start, stop

**FarcasterMessageService:**
- Implements IMessageService
- Methods: getMessages, getMessage, sendMessage

**FarcasterCastService:**
- Implements IPostService
- Methods: getCasts, createCast, deleteCast, likeCast, unlikeCast, recast, unrecast

### Client Architecture (FarcasterClient)
Core wrapper around Neynar API:
- `publishCast` with embeds and reply options
- `reply` method for responding
- `deleteCast` functionality
- User operations: getUser, getUserByFid, getUserByUsername
- Timeline operations: getMentions, getTimeline, getCast
- Engagement operations: like, recast, follow

### Utility Functions
- `castUuid` - Generate unique cast identifiers
- `neynarCastToCast` - Format conversion
- `formatCastTimestamp` - Timestamp formatting
- `formatCast` - AI processing preparation
- `formatTimeline` - Context preparation
- `lastCastCacheKey` - Cache key generation

### AsyncQueue
Manages async operations with single concurrency, preventing rate limiting.

### Caching Strategy
| Cache | TTL | Max Entries |
|-------|-----|-------------|
| Cast Cache | 30 minutes | 9,000 |
| Profile Cache | - | User data |
| Timeline Cache | - | Recent casts |
| Last Cast Tracking | - | Per-agent timestamps |

### Rate Limiting
- Exponential backoff for API requests
- Hub limits: ~100 req/min
- Cache frequently accessed data

### Security
- Store credentials in environment variables
- Never commit keys to version control
- Separate dev/prod keys
- Validate cast length (320 char max)
- Sanitize user inputs
- Verify message signatures

### Migration Note (v2)
v2 uses named imports and character-based configuration instead of class instantiation.

### License
MIT

---

## Cross-Cutting Notes

### Embedding Models and Dimensions
The documentation references these embedding models but does not explicitly state embedding dimensions:
- `text-embedding-3-large` (OpenAI) - primary recommended
- `text-embedding-3-small` (OpenAI) - mentioned in config examples
- `qwen3-embedding`
- `gemini-embedding`
- `mistral-embed`

Embeddings are generated via `runtime.useModel(ModelType.TEXT_EMBEDDING, { text })`.

### Knowledge Ingestion Pipeline Summary
1. **Input**: File upload (base64), direct text, character knowledge array, or auto-load from `docs/` folder
2. **Extraction**: Format-specific parsers (PDF, DOCX, TXT, MD, JSON, CSV)
3. **Deduplication**: Content-based ID generation using content hash + agentId + filename + contentType (first 2000 chars)
4. **Chunking**: 500-token chunks with 100-token overlap, smart boundary detection using separators `['\n\n', '\n', '. ', ' ']`
5. **Contextual Enrichment** (optional): LLM adds 60-200 tokens of context per chunk
6. **Embedding**: Batch processing (10 chunks/batch, 1s delay), using configured embedding model
7. **Storage**: Documents table (full content + metadata) + Knowledge table (chunk embeddings + position metadata)
8. **Retrieval**: Vector similarity search, threshold 0.7, provider injects up to 5 fragments (~4000 tokens) per message

### Platform Plugin Pattern
All platform plugins (Discord, Twitter, Farcaster) follow:
- Service class extending `Service` with initialize/start/stop lifecycle
- Manager classes for specific concerns (messages, posting, interactions)
- Provider functions for context injection
- Action definitions for agent capabilities
- Event handlers for platform events
- Caching with LRU eviction
- Rate limiting with exponential backoff
