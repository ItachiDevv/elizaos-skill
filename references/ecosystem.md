# ElizaOS Ecosystem & GitHub Organization Reference

Organization: https://github.com/elizaOS (verified 2026-06-29; 16 public repositories visible via unauthenticated API — previous count of 64 likely included private repos or repos since removed/transferred)

## Core Framework

| Repository | Stars | Description |
|-----------|-------|-------------|
| **eliza** | 18.7k | Main monorepo — v2 beta (`develop` branch, `2.0.4`). 36 packages including `agent, app, app-core, core, cloud, deploy, elizaos, native, os, prompts, shared, skills, tui, ui, vault` plus 151 first-party plugins under top-level `plugins/`. MIT licensed. |
| **docs** | 4 | Official documentation source (MDX/Mintlify). Hosted at docs.elizaos.ai |
| **roadmap** | 15 | Project roadmap and planning |

## Starters & Templates

| Repository | Stars | Description |
|-----------|-------|-------------|
| **eliza-nextjs-starter** | 30 | v2 Next.js document chat demo. Best starting point for v2 web apps. |
| **eliza-3d-hyperfy-starter** | 40 | 3D MMO agent with Hyperfy plugin. For game/metaverse prototyping. |
| **eliza-plugin-starter** | 38 | Plugin development starter (Solana hackathon origin). |
| **autonomous-starter** | 26 | Autonomous agent starter project. |
| **eliza-starter** | 372 | ARCHIVED — v1 starter. Use eliza-nextjs-starter for v2. |
| **sandbox-template-cloud** | 3 | Eliza Cloud sandbox app template. |

## Showcase Agents

| Repository | Stars | Description |
|-----------|-------|-------------|
| **spartan** | 80 | Quantitative trading agent. |
| **otaku** | 23 | DeFi trading and research agent (JavaScript). |
| **otc-agent** | 8 | OTC trading agent. |
| **SWEagent** | 20 | Software engineering agent (TypeScript). |
| **prr** | 3 | PR review agent — "sits on your PR and won't get up until it's ready." |
| **elizas-world** | 37 | Agent swarm showcase — "Witness the swarm awaken." |
| **the-org** | 51 | Multi-agent organization template for team-based agent deployments. |

## Data & Knowledge

| Repository | Stars | Description |
|-----------|-------|-------------|
| **knowledge** | 61 | Ecosystem data pipeline: news, GitHub updates, discussion summaries. Python-based, feeds RAG systems. |
| **characters** | 45 | Collection of character files for agents. |
| **characterfile** | 383 | Standard format specification for character data in agent/LLM contexts. |
| **awesome-eliza** | 93 | Curated list of ElizaOS resources, plugins, tutorials, and community projects. |

## Infrastructure & Tools

| Repository | Stars | Description |
|-----------|-------|-------------|
| **autofun-idl** | 3 | Anchor IDLs for the auto.fun launchpad. The auto.fun product itself is NOT in this GitHub org — it lives at autofun.tech / auto.fun (closed-source or hosted elsewhere). |
| **elizaos.github.io** | 104 | Contributor leaderboard website. |
| **registry** | 21 | elizaOS plugin registry. |
| **plugins-automation** | 7 | Automation scripts for the 150+ plugins in the eliza-plugins org. |
| **mcp-gateway** | 13 | MCP gateway service. |
| **openclaw-adapter** | 40 | Run Eliza plugins inside OpenClaw — wallets, connectors, services. |
| **x402.elizaos.ai** | 3 | Dynamic x402 routing with intelligent content negotiation. |
| **agentmemory** | 236 | Easy-to-use agent memory, powered by chromadb and postgres. |
| **discord-summarizer** | 94 | Use LLMs to summarize discord channels into actionable insights. |
| **LiveVideoChat** | 75 | Live video chat infrastructure. |
| **cloud** / **cloud-mini-apps** / **eliza-app** / **mobile** | — | Eliza Cloud + native app surfaces. |
| **agentshell / agentbrowser / agentcomms / agentlogger / agentaction / agentagenda / agentloop** | — | Modular standalone agent components — shell, browser, comms, logger, action chaining, task manager, run loop. |
| **discord-summarizer** | 88 | LLM-powered Discord channel summarization. |
| **mcp-gateway** | 11 | MCP (Model Context Protocol) gateway. |
| **registry** | 21 | ARCHIVED — Original plugin registry. |
| **plugins-automation** | 7 | Scripts to manage 150+ plugins in eliza-plugins org. |
| **mobile** | 1 | React Native mobile app with Privy auth. |
| **brandkit** | 19 | Official logos, assets, and design resources. |

## Python Agent Toolkit

A suite of composable Python libraries for building agents:

| Repository | Stars | Description |
|-----------|-------|-------------|
| **agentmemory** | 230 | Agent memory with chromadb and postgres. |
| **agentbrowser** | 23 | Browser automation for agents. |
| **agentshell** | 17 | Shell/terminal access for agents. |
| **agentagenda** | 22 | Task management for agents. |
| **agentcomms** | 18 | Communication connectors for agents. |
| **agentlogger** | 10 | Colorful terminal logging. |
| **agentloop** | 14 | Simple start/stop loop with step-through. |
| **agentaction** | 13 | Action chaining and history. |
| **easycompletion** | 18 | Easy OpenAI text completion and function calling. |

## Other Repos

| Repository | Stars | Description |
|-----------|-------|-------------|
| **LiveVideoChat** | 75 | Video chat application. |
| **LJSpeechTools** | 26 | Tools for making LJSpeech datasets. |
| **classified** | 22 | "Nothing to see here." |
| **trust_scoreboard** | 11 | Trust scoring system. |
| **aum-tracker** | 12 | AUM (Assets Under Management) tracking. |
| **elizas-list** | 8 | Community project directory. |
| **hat** / **hats** | 3/3 | Hat protocol and image tools. |
| **.cursor** | 13 | Cursor IDE rules and config for ElizaOS development. |
| **x402.elizaos.ai** | 2 | Dynamic x402 routing with content negotiation. |
| **discrub-ext** | 3 | Discord message manipulation Chrome/Firefox extension. |
| **character-migrator** | 1 | Character file migration tool (v1 -> v2). |
| **plugin-specification** | 1 | Plugin specification document. |
| **vercel-api** | 1 | Next.js Vercel API routes. |

## Related Organization

**eliza-plugins** — Separate GitHub organization housing 150+ community plugins managed via the plugins-automation repo.

## Key Resources

- **Docs**: https://docs.elizaos.ai
- **Website**: https://elizaos.ai
- **GitHub**: https://github.com/elizaos
- **Paper**: "Eliza: A Web3 friendly AI Agent Operating System" (arXiv:2501.06781)
- **Full doc index**: https://docs.elizaos.ai/llms.txt
