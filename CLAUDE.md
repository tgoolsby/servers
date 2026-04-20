# Claude Code — MCP Servers Repository

## Project Overview

This is the **Model Context Protocol (MCP) reference servers** repository by Anthropic. It contains official reference implementations demonstrating how to expose tools, resources, and prompts to LLMs via MCP.

Each server lives under `src/<server-name>/` and is a standalone npm workspace package. Servers are implemented in **TypeScript** (primary) or **Python**.

## Architecture

```
src/
├── brave-search/         # Web search via Brave API
├── filesystem/           # Secure file operations
├── git/                  # Git repo tools (Python)
├── github/               # GitHub API integration
├── memory/               # Knowledge graph persistence
├── postgres/             # Read-only DB access
├── puppeteer/            # Browser automation
├── sequentialthinking/   # Reflective reasoning tool
├── slack/                # Slack messaging
├── sqlite/               # SQLite interaction
└── ...                   # More reference servers
```

## Common Commands

```bash
# Install all dependencies
npm install

# Build all servers
npm run build

# Build a specific server
npm run build --workspace=src/filesystem

# Watch mode for active development
npm run watch --workspace=src/<name>

# Run a server directly (after build)
node src/filesystem/dist/index.js /allowed/path

# Test interactively with MCP Inspector
npx @modelcontextprotocol/inspector node src/<name>/dist/index.js
```

## Development Workflow

1. Make changes in `src/<server-name>/index.ts`
2. `npm run build --workspace=src/<server-name>` to compile
3. Test with MCP Inspector or Claude Code directly
4. Each server's `README.md` lists its tools, resources, and required env vars

## Key Conventions

- Each server uses `@modelcontextprotocol/sdk` — never raw HTTP
- Entry point is always `src/<name>/index.ts` (TypeScript) or `src/<name>/__init__.py` (Python)
- Build output: `src/<name>/dist/`
- Input validation uses Zod schemas matching the MCP JSON Schema spec
- ES modules (`"type": "module"`) — use `.js` extensions in all imports

## Adding a New Server

1. `mkdir src/<name>` and add `package.json` with `"name": "@modelcontextprotocol/server-<name>"`
2. Implement using the TypeScript SDK pattern below
3. Add to root `package.json` workspaces array and dependencies
4. Add a `README.md` documenting tools/resources/env vars
5. Update root `README.md`

## TypeScript Server Pattern

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  { name: "my-server", version: "0.1.0" },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "my_tool",
      description: "Does something useful",
      inputSchema: {
        type: "object",
        properties: { query: { type: "string" } },
        required: ["query"],
      },
    },
  ],
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;
  if (name === "my_tool") {
    // tool logic here
    return { content: [{ type: "text", text: "result" }] };
  }
  throw new Error(`Unknown tool: ${name}`);
});

const transport = new StdioServerTransport();
await server.connect(transport);
```

## Environment Variables

API keys are passed via environment variables — never hardcoded:
- `BRAVE_API_KEY` — brave-search
- `GITHUB_PERSONAL_ACCESS_TOKEN` — github
- `SLACK_BOT_TOKEN`, `SLACK_TEAM_ID` — slack
- `POSTGRES_CONNECTION_STRING` — postgres

## Claude Code Setup for This Repo

### MCP Servers (configured in `.claude/settings.json`)

The following servers are pre-built and wired for use within Claude Code sessions:
- **filesystem** — read/write files scoped to this repo
- **memory** — persist knowledge graph notes across sessions
- **sequential-thinking** — structured multi-step reasoning

To rebuild them: `npm run build --workspace=src/filesystem` (etc.)

### Useful Claude Code Slash Commands

| Command | Purpose |
|---|---|
| `/help` | Show all available commands |
| `/clear` | Clear conversation context |
| `/compact` | Compress context while preserving key info |
| `/cost` | Show token usage for the session |
| `/review` | Code review of current changes |
| `/init` | Regenerate CLAUDE.md from codebase scan |

### Claude Code Tips for This Repo

- **Ask Claude to build before testing**: "build the filesystem server and run the inspector"
- **Use memory server** to persist notes about server internals across sessions
- **Sequential thinking** helps when designing new tool schemas — ask Claude to think through edge cases step by step
- **Reference existing servers** when adding new ones: "model this after src/sqlite/index.ts"
- **Hooks**: Add pre/post tool hooks in `.claude/settings.json` to auto-run linters or tests on save

### Adding Claude Code Hooks (`.claude/settings.json`)

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npm run build --workspace=src/$(git diff --name-only HEAD | grep '^src/' | cut -d/ -f2 | head -1) 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```
