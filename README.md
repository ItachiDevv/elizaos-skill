# ElizaOS Skill for Claude Code

A comprehensive [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) that provides expert-level knowledge of the [ElizaOS](https://elizaos.ai) framework for building autonomous AI agents.

## What This Skill Does

When installed, Claude Code gains deep knowledge of ElizaOS including:

- **Architecture** -- runtime, plugins, memory, events, database (Drizzle ORM + PGLite/PostgreSQL)
- **Plugin Development** -- actions, providers, evaluators, services, routes, schemas, events
- **Platform Integrations** -- Discord, Telegram, Twitter, Farcaster, Solana, EVM, MCP
- **LLM Providers** -- OpenAI, Anthropic, Ollama, OpenRouter, Google GenAI
- **Application Patterns** -- embedded runtime, REST API, dynamic context injection, dual AI fallback
- **v2.0.0 Alpha** -- Python/Rust SDKs, protobuf schemas, capability tiers, autonomy system
- **Deployment** -- Eliza Cloud, Docker, TEE (Phala)
- **CLI** -- project scaffolding, dev mode, plugin management, testing

## Installation

### From GitHub (recommended)

```bash
claude skill install --from https://github.com/ItachiDevv/elizaos-skill
```

### Manual

Copy `SKILL.md` and the `references/` folder to `~/.claude/skills/elizaos/`:

```bash
mkdir -p ~/.claude/skills/elizaos/references
cp SKILL.md ~/.claude/skills/elizaos/
cp references/*.md ~/.claude/skills/elizaos/references/
```

## Skill Structure

```
elizaos-skill/
  SKILL.md                              # Main skill (loaded into context)
  references/
    api-reference.md                    # REST API + WebSocket endpoints
    ecosystem.md                        # 57 GitHub repos, starters, tools
    integration-patterns.md             # Embedded/external, lifecycle, fallback
    platform-integrations.md            # All platform + provider configs
    plugin-development.md               # Full plugin authoring guide
    v2-architecture.md                  # v2.0.0 types, runtime, breaking changes
  sources/                              # Raw documentation extractions (reference only)
```

## Coverage

| Topic | Detail Level |
|-------|-------------|
| Core Architecture | Full runtime, pipeline, memory, events |
| Plugin System | Complete interfaces, patterns, Drizzle schemas |
| v1.x Stable (develop) | Production-ready, `@elizaos/core` v1.7.2 |
| v2.0.0 Alpha | Full type system, breaking changes, SDKs |
| Platforms (6) | Discord, Telegram, Twitter, Farcaster, Solana, EVM |
| LLM Providers (5) | OpenAI, Anthropic, Ollama, OpenRouter, Google |
| REST API | All endpoints, WebSocket, Socket.IO |
| Database | PGLite, PostgreSQL, Drizzle ORM, plugin schemas |
| Deployment | Eliza Cloud, Docker, TEE |
| Integration Patterns | Embedded, REST, context injection, lifecycle |
| Ecosystem | 57 repos, starters, Python toolkit |

## Accuracy

Content was extracted from:
- [docs.elizaos.ai](https://docs.elizaos.ai) (official documentation)
- [github.com/elizaOS/eliza](https://github.com/elizaOS/eliza) source code (develop + v2.0.0 branches)
- npm `@elizaos/core` package analysis

Last verified: February 2026. Stable: v1.7.2. Alpha: v2.0.0-alpha.10.

## Contributing

PRs welcome. When updating content:

1. Verify against the official [ElizaOS docs](https://docs.elizaos.ai) and [source code](https://github.com/elizaOS/eliza)
2. Keep SKILL.md as a concise overview; put details in `references/`
3. Reference files are loaded on-demand by Claude -- keep them focused

## License

MIT
