# Nativ Documentation

Public-facing documentation, setup guides, and embeddable assets for [Nativ](https://usenativ.com) integrations.

## Contents

| Directory | Description |
|---|---|
| [`custom-gpt/`](custom-gpt/) | OpenAPI spec, system prompt, and step-by-step guide for publishing a Custom GPT that connects to Nativ |
| [`openai-assistants/`](openai-assistants/) | Function-calling tool definitions (`tools.json`) and setup guide for the OpenAI Assistants API |
| [`integrations-embed/`](integrations-embed/) | Embeddable HTML integrations page used on [usenativ.com](https://usenativ.com) |

## Related Repos

| Repo | Install | Description |
|---|---|---|
| [nativ-python](https://github.com/Nativ-Technologies/nativ-python) | `pip install nativ` | Python SDK |
| [nativ-node](https://github.com/Nativ-Technologies/nativ-node) | `npm install nativ-sdk` | Node.js / TypeScript SDK |
| [nativ-mcp](https://github.com/Nativ-Technologies/nativ-mcp) | `pip install nativ-mcp` | MCP server for Claude, Cursor, etc. |
| [langchain-nativ](https://github.com/Nativ-Technologies/langchain-nativ) | `pip install langchain-nativ` | LangChain tools |
| [nativ-figma-plugin](https://github.com/Nativ-Technologies/nativ-figma-plugin) | Figma Community | Figma plugin |

## Getting Started

All integrations authenticate with a Nativ API key (`nativ_xxx...`).  
Get one at **[dashboard.usenativ.com](https://dashboard.usenativ.com) → Settings → API Keys**.

## License

MIT
