# Nativ Documentation

Public-facing documentation, setup guides, and embeddable assets for [Nativ](https://usenativ.com) integrations.

## Start with the workflow

- [Current product and installation catalogue](https://www.usenativ.com/integrations/): distinguish published plugins, available developer interfaces, dashboard connections and early access.
- [Build with Nativ](https://www.usenativ.com/integrations/build/): Python/TypeScript quickstarts and custom-plugin architecture.
- [MCP setup](https://www.usenativ.com/integrations/mcp/): local/hosted setup, tool workflows and media inputs.
- [Full MCP capability mapping](https://github.com/Nativ-Technologies/nativ-mcp/blob/main/docs/capabilities.md): exact text, image, audio, subtitle and workspace operations.
- [Existing public custom GPT](https://chatgpt.com/g/g-69979fd5b5a48191aec8c254b15014e4-nativ-ai-localization): conversational text localisation, distinct from the full MCP server.

The setup files below include historical publisher examples. They are not a
universal capability, pricing or authentication contract. Use current public
schemas and the relevant SDK version when implementing a connector.

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

Authentication depends on the interface:

| Interface | Connection |
| --- | --- |
| Full MCP, JSON tools and SDK/API integrations | Nativ API key; keep application keys on your server |
| Focused `/chatgpt/mcp` endpoint | OAuth sign-in; seven focused tools, distinct from full MCP’s 51 |
| Existing custom GPT | Its own GPT Actions account connection |
| Dashboard and creative plugins | The account/host sign-in described by that integration |

Create API keys at **[dashboard.usenativ.com](https://dashboard.usenativ.com) →
Settings → API Keys**. Never put real keys in committed examples or chat
messages. A reachable OAuth endpoint does not establish public plugin-directory
approval. Generation can use the connected Nativ account’s credits; check
current workspace usage rather than relying on old fixed-price examples.

## License

MIT
