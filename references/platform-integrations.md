# Platform & Provider Integrations Reference

## LLM Provider System

### Model Priority Routing

ElizaOS uses a **priority-based model routing system**. Multiple plugins can register handlers for the same `ModelType`. The handler with the highest priority wins. If it fails, the next-highest is tried (fallback chain).

**Default plugin loading order**: Anthropic → OpenRouter → OpenAI → Google GenAI → Ollama

This means if both Anthropic and OpenAI are configured, Anthropic handles TEXT_SMALL/TEXT_LARGE, while OpenAI handles TEXT_EMBEDDING (since Anthropic has none).

### OpenAI
```bash
elizaos plugins add @elizaos/plugin-openai
```
```env
OPENAI_API_KEY=sk-...
OPENAI_SMALL_MODEL=gpt-4o-mini
OPENAI_LARGE_MODEL=gpt-4o
OPENAI_EMBEDDING_MODEL=text-embedding-3-small   # 1536 dimensions
```
Provides: TEXT_SMALL, TEXT_LARGE, TEXT_EMBEDDING, IMAGE, TRANSCRIPTION, TEXT_TO_SPEECH, OBJECT_SMALL, OBJECT_LARGE

### Anthropic
```bash
elizaos plugins add @elizaos/plugin-anthropic
```
```env
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_SMALL_MODEL=claude-3-haiku-20240307
ANTHROPIC_LARGE_MODEL=claude-3-5-sonnet-latest
```
**CRITICAL**: Anthropic has NO embedding model. You MUST include OpenAI or Ollama alongside Anthropic for embeddings. Without an embedding provider, memory search, knowledge features, and fact extraction will all fail silently.

### Ollama (Local)
```bash
elizaos plugins add @elizaos/plugin-ollama
```
```env
OLLAMA_API_ENDPOINT=http://localhost:11434/api
OLLAMA_SMALL_MODEL=llama3.2
OLLAMA_LARGE_MODEL=llama3.1:70b
OLLAMA_EMBEDDING_MODEL=nomic-embed-text          # 768 dimensions
```
Requires: `ollama pull <model>` before use. Embedding dimensions vary by model.

### OpenRouter
```bash
elizaos plugins add @elizaos/plugin-openrouter
```
Multi-provider routing with caching (90% cost reduction). Supports Anthropic, OpenAI, and other models behind a single API key.

### Google GenAI
```bash
elizaos plugins add @elizaos/plugin-google-genai
```
Gemini models for text, image, and multimodal tasks.

## Communication Platforms

### Discord
```bash
elizaos plugins add @elizaos/plugin-discord
```
```env
DISCORD_APPLICATION_ID=your_app_id
DISCORD_API_TOKEN=your_bot_token
CHANNEL_IDS=optional_channel_filter          # Comma-separated
DISCORD_VOICE_CHANNEL_ID=optional_voice
```
Features: Text messages, voice channels, slash commands, media processing, server sync.
Event handlers: MESSAGE_RECEIVED, VOICE_MESSAGE_RECEIVED, WORLD_JOINED, ENTITY_JOINED.

### Twitter/X
```bash
elizaos plugins add @elizaos/plugin-twitter
```
```env
TWITTER_API_KEY=oauth_api_key
TWITTER_API_SECRET_KEY=oauth_secret
TWITTER_ACCESS_TOKEN=access_token
TWITTER_ACCESS_TOKEN_SECRET=access_secret
TWITTER_POST_ENABLE=true
TWITTER_SEARCH_ENABLE=true
TWITTER_DRY_RUN=false                        # Set true to test without posting
```
Features: Posting tweets, timeline monitoring, intelligent engagement, thread management.

### Telegram
```bash
elizaos plugins add @elizaos/plugin-telegram
```
```env
TELEGRAM_BOT_TOKEN=bot_token_from_botfather
TELEGRAM_API_ROOT=optional_custom_api          # Custom Telegram API server
TELEGRAM_ALLOWED_CHATS='["chatId1","chatId2"]' # JSON array (NOT comma-separated)
```
Features: Messages, media, inline keyboards, group management, bot commands.

Uses Telegraf client library. Bot must be created via @BotFather. For group chats, bot must be added as admin. `TELEGRAM_ALLOWED_CHATS` accepts a JSON-stringified array of chat IDs to restrict which chats the bot responds in.

### Farcaster
```bash
elizaos plugins add @elizaos/plugin-farcaster
```
Features: Casting, engagement, social graph integration, Warpcast support.

## Blockchain / DeFi

### Solana
```bash
elizaos plugins add @elizaos/plugin-solana
```
```env
SOLANA_PRIVATE_KEY=base58_private_key
SOLANA_PUBLIC_KEY=optional_for_readonly
SOLANA_RPC_URL=https://api.mainnet-beta.solana.com
HELIUS_API_KEY=optional_for_enhanced_rpc
BIRDEYE_API_KEY=optional_for_market_data
```
Features: SOL/SPL transfers, Jupiter swaps, portfolio tracking, price feeds, DeFi operations.

### EVM (Ethereum, Polygon, Base, Arbitrum, Optimism, 30+ chains)
```bash
elizaos plugins add @elizaos/plugin-evm
```
```env
EVM_PRIVATE_KEY=hex_private_key
ETHEREUM_PROVIDER_MAINNET=https://eth-mainnet.g.alchemy.com/v2/KEY
ETHEREUM_PROVIDER_BASE=https://base-mainnet.g.alchemy.com/v2/KEY
```
Features: Multi-chain token transfers, swaps (Uniswap/1inch), bridging, governance voting, contract interaction.

## Knowledge / RAG

### Knowledge Plugin
```bash
elizaos plugins add @elizaos/plugin-knowledge
```
```env
CTX_KNOWLEDGE_ENABLED=true                   # Contextual embeddings (50% better retrieval)
LOAD_DOCS_ON_STARTUP=true                    # Auto-load from agent's docs folder
EMBEDDING_PROVIDER=openai                    # Default provider
TEXT_EMBEDDING_MODEL=text-embedding-3-small
```

#### Document Ingestion
Supports: PDF, DOCX, TXT, MD, CSV, URLs.
Pipeline: Extract text -> Chunk (500 tokens, 100 overlap) -> Contextual enrichment -> Embed -> Store.
Deduplication: Content hash + agent ID + filename prevents duplicates.

#### REST Endpoints
- `POST /knowledge/upload` — Multipart file upload
- `GET /knowledge/documents` — List with pagination
- `DELETE /knowledge/documents/{id}` — Remove document + embeddings
- `GET /knowledge/search?query=...&limit=5` — Semantic search

#### Programmatic Access
```typescript
const knowledge = runtime.getService('knowledge');
await knowledge.addKnowledge({
  content: documentContent,
  originalFilename: 'guide.pdf',
  contentType: 'application/pdf',
});
```

#### Character Knowledge
```typescript
const character: Character = {
  knowledge: [
    'Inline text knowledge',
    { path: './docs/handbook.md', shared: false },
  ],
};
```

## SQL / Database

### SQL Plugin (Required — priority 0)
```bash
elizaos plugins add @elizaos/plugin-sql
```
```env
# PostgreSQL (production)
POSTGRES_URL=postgresql://user:pass@host:5432/database

# PGLite (local development — no external DB needed)
PGLITE_DATA_DIR=/path/to/db    # Defaults to ./.eliza/.elizadb
```

**Architecture**:
- `priority: 0` ensures it loads before all other plugins
- Calls `runtime.registerDatabaseAdapter(dbAdapter)` during init
- Adapter selection: `POSTGRES_URL` set → `PgDatabaseAdapter`, else → `PgliteDatabaseAdapter`
- Both extend `BaseDrizzleAdapter` with default **384-dimension** embeddings
- Provides: Drizzle ORM integration, automatic schema migrations, IDatabaseAdapter
- `DatabaseMigrationService` handles all plugin schema discovery and migration
- Connection managers are singletons (`PGliteClientManager`, `PostgresConnectionManager`)

**Core tables auto-created**: `agents`, `memories`, `entities`, `relationships`, `rooms`, `participants`, `messages`, `embeddings`, `cache`, `logs`, `tasks`

**Compatible PostgreSQL providers**: Supabase, Neon, RDS, Cloud SQL, self-hosted (v12+)

```
# Connection string examples
Supabase:  postgresql://postgres:[password]@[project].supabase.co:5432/postgres
Neon:      postgresql://[user]:[password]@[project].neon.tech/[database]?sslmode=require
```

## Other Service Plugins

| Plugin | Service Type | Purpose |
|--------|-------------|---------|
| @elizaos/plugin-openai | TRANSCRIPTION | Audio-to-text (Whisper) |
| @elizaos/plugin-video | VIDEO | Video processing |
| @elizaos/plugin-browser | BROWSER | Web automation |
| @elizaos/plugin-pdf | PDF | Document processing |
| @elizaos/plugin-s3 | REMOTE_FILES | AWS S3 cloud storage |
| @elizaos/plugin-web-search | WEB_SEARCH | Web search queries |
| @elizaos/plugin-email | EMAIL | Email sending/receiving |

## MCP (Model Context Protocol) Support

ElizaOS supports external tools via MCP:

### STDIO Servers (Local processes)
```json
{
  "mcpServers": {
    "firecrawl": {
      "command": "npx",
      "args": ["-y", "firecrawl-mcp"]
    }
  }
}
```

### SSE Servers (Remote HTTP)
```json
{
  "mcpServers": {
    "remote-tools": {
      "url": "https://api.example.com/mcp/sse"
    }
  }
}
```
