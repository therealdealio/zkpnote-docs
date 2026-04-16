# ZKPnote MCP Server

The ZKPnote MCP (Model Context Protocol) server lets AI agents like Claude read, write, search, and verify notes in your encrypted vault. Your seed phrase stays local — all encryption happens on your machine before anything touches the network.

**Live deployment:** [www.zkpnote.com](https://www.zkpnote.com) (Solana devnet)

## What You Can Do

Once connected, ask Claude things like:

- "Save this research as a note called 'AI Safety Summary'"
- "List my notes" / "Read my note about Solana"
- "Save these five meeting summaries to my Meetings folder" (batched — one vault sync)
- "Check if this text has been proved on-chain"
- "Find similar proved content"
- "Search my notes for anything about pricing"
- "Build a knowledge graph of all my notes"

## Prerequisites

- **Node.js 18+** — [nodejs.org](https://nodejs.org)
- **A ZKPnote account** — Create one at [www.zkpnote.com](https://www.zkpnote.com) using a seed phrase (not Phantom)
- **Claude Desktop** or **Claude Code** — The MCP server works with any MCP-compatible client

> **Important:** The MCP server uses your **seed phrase** for encryption. If you created your vault with Phantom, you'll need to create a separate seed-phrase vault or use Phantom for browser and seed phrase for MCP.

## Quick Start

### 1. Clone and build

```bash
git clone https://github.com/therealdealio/zkpnote.git
cd zkpnote/packages/mcp-server
npm install
npm run build
```

This compiles the TypeScript server to `build/index.js`.

### 2. Configure your MCP client

Choose your client below and add the configuration.

#### Claude Desktop

Edit `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows):

```json
{
  "mcpServers": {
    "zkpnote": {
      "command": "node",
      "args": ["/absolute/path/to/zkpnote/packages/mcp-server/build/index.js"],
      "env": {
        "ZKPNOTE_SEED_PHRASE": "your twelve word seed phrase goes here",
        "ZKPNOTE_API_URL": "https://www.zkpnote.com"
      }
    }
  }
}
```

#### Claude Code (CLI)

Edit `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "zkpnote": {
      "command": "node",
      "args": ["/absolute/path/to/zkpnote/packages/mcp-server/build/index.js"],
      "env": {
        "ZKPNOTE_SEED_PHRASE": "your twelve word seed phrase goes here",
        "ZKPNOTE_API_URL": "https://www.zkpnote.com"
      }
    }
  }
}
```

#### Claude Code (VS Code Extension)

Open Settings (`Cmd+,`) > search "Claude MCP" > edit `settings.json`, or add to your project's `.claude/settings.json`:

```json
{
  "mcpServers": {
    "zkpnote": {
      "command": "node",
      "args": ["/absolute/path/to/zkpnote/packages/mcp-server/build/index.js"],
      "env": {
        "ZKPNOTE_SEED_PHRASE": "your twelve word seed phrase goes here",
        "ZKPNOTE_API_URL": "https://www.zkpnote.com"
      }
    }
  }
}
```

### 3. Restart your client

- **Claude Desktop:** Quit and reopen
- **Claude Code CLI:** Start a new session
- **VS Code:** Run "Claude: Manage MCP Servers" from the command palette and restart the server

### 4. Verify

Ask Claude: **"List my ZKPnote notes"**. You should see your vault contents.

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `ZKPNOTE_SEED_PHRASE` | Yes* | — | Your 12-word BIP-39 seed phrase |
| `ZKPNOTE_API_URL` | No | `https://zkpnote.com` | ZKPnote API URL (use `https://www.zkpnote.com` for devnet) |

\* Verify and search tools work without a seed phrase (read-only access to public proofs).

## Available Tools (18)

### Vault Operations

| Tool | Description |
|------|-------------|
| `save_note` | Save a markdown note (encrypted E2E before storage) |
| `save_notes` | Batch save multiple notes in one vault sync |
| `list_notes` | List all notes with titles, folders, proof status, timestamps |
| `read_note` | Read a note by title (partial match) or exact ID |
| `update_note` | Update a note's title or content |
| `delete_note` | Delete a note by title or ID |
| `reorder_notes` | Set custom sort order for notes in a folder |
| `list_folders` | List all folders with note counts |

### Proofs (Solana On-Chain)

| Tool | Description |
|------|-------------|
| `verify_content` | Check if content has been proved on Solana |
| `search_similar` | Find similar proved content via fuzzy trigram matching |
| `list_proofs` | List all on-chain proofs for the connected wallet |
| `recover_note` | Recover lost note content from a proof transaction signature |

### Marketplace

| Tool | Description |
|------|-------------|
| `browse_marketplace` | Browse/search listings with category and keyword filters |
| `get_listing` | Get full details of a specific listing |
| `cancel_listing` | Cancel one of your own listings |
| `marketplace_analytics` | Sales stats and transaction history |

### Knowledge Graph

| Tool | Description |
|------|-------------|
| `build_knowledge_graph` | Index all notes into a searchable graph with summaries, tags, and relationships |
| `query_knowledge_graph` | Find relevant notes by keyword, project, or tag without reading full content |

## Remote HTTP Transport (Claude mobile / claude.ai web)

Claude on iOS/Android and at claude.ai only accept **remote** connectors — they can't spawn a local stdio process. ZKPnote ships a matching remote transport at `POST /api/mcp` that exposes the same tool handlers as the stdio server, backed by the same encrypted vault in Supabase.

**Source:** `src/lib/mcpServer.ts` (factory `buildMcpServer({ seedPhrase, apiUrl })`) wired to `src/app/api/mcp/route.ts` via `WebStandardStreamableHTTPServerTransport` from `@modelcontextprotocol/sdk`.

### Endpoint

```
POST https://www.zkpnote.com/api/mcp
Authorization: Bearer <ZKPNOTE_MCP_TOKEN>
Content-Type: application/json
Accept: application/json, text/event-stream

<JSON-RPC 2.0 body>
```

Mobile clients that can't set headers may pass the token as a query parameter instead: `?token=<ZKPNOTE_MCP_TOKEN>`.

> Always use the `www.` subdomain. The apex `zkpnote.com` issues a 307 redirect to `www`, and most HTTP clients drop the `Authorization` header across the redirect — your request will return `Unauthorized: missing bearer token`.

### Deployment requirements

The remote endpoint is currently **single-tenant**: one seed phrase per Vercel deployment, gated by one shared-secret token. To run your own:

1. Fork [`therealdealio/zkpnote`](https://github.com/therealdealio/zkpnote) and deploy to Vercel.
2. Set three env vars (Production + Preview scopes):
   - `ZKPNOTE_SEED_PHRASE` — the 12-word BIP-39 phrase for the vault this endpoint serves
   - `ZKPNOTE_MCP_TOKEN` — a long random secret (`openssl rand -hex 32`)
   - `ZKPNOTE_API_URL` = `https://www.zkpnote.com` (or your own deployment URL)
3. Redeploy. The endpoint fails closed — if `ZKPNOTE_MCP_TOKEN` is unset, `POST /api/mcp` returns HTTP 500 `Server misconfigured` rather than silently accepting unauthenticated requests.

### End-to-end test

The repo ships `scripts/test-mcp-bot.mjs` — 26 assertions exercising tool discovery, CRUD, batch save, folder creation, and cleanup, marker-isolated so it leaves the vault in its original state.

```bash
MCP_URL=https://www.zkpnote.com/api/mcp \
MCP_TOKEN=your_long_random_token \
node scripts/test-mcp-bot.mjs
```

Preview deployments gated by Vercel Deployment Protection: add `VERCEL_BYPASS=<protection-bypass-secret>` as well and the bot will forward it as `x-vercel-protection-bypass`.

### Available Tools (Remote — 15)

Same set as stdio with the three marketplace-detail tools omitted (`get_listing`, `cancel_listing`, `marketplace_analytics`). Planned for a follow-up release.

| Tool | Vault | Proofs | Market | KG |
|------|:-----:|:-----:|:-----:|:-----:|
| `save_note`, `save_notes`, `list_notes`, `read_note`, `update_note`, `delete_note`, `reorder_notes`, `list_folders` | ✓ | | | |
| `verify_content`, `search_similar`, `list_proofs`, `recover_note` | | ✓ | | |
| `browse_marketplace` | | | ✓ | |
| `build_knowledge_graph`, `query_knowledge_graph` | | | | ✓ |

### Claude client setup (remote)

**Claude mobile (iOS / Android):**
1. Settings → Connectors → **Add custom connector**
2. Name: `ZKPnote`
3. URL: `https://www.zkpnote.com/api/mcp?token=YOUR_TOKEN`
4. Enable for chat, then ask "What tools do you have from ZKPnote?"

**claude.ai web:** Settings → Connectors → **Add custom connector** → same URL.

## Security Model

### Stdio transport
- **Seed phrase never leaves your machine.** It's used only by the local MCP server process to derive encryption keys.
- **All note content is encrypted with XChaCha20-Poly1305** before being sent to the ZKPnote API.
- **The server communicates over HTTPS** with the ZKPnote backend.
- **No data is written to disk** by the MCP server — everything runs in memory.
- **The API URL is the only external endpoint** the server contacts.

### Remote HTTP transport
- **Seed phrase is server-side** (Vercel env var) — you're trusting your hosting provider with key material the way any self-hosted app with a master secret does.
- **Bearer token (`ZKPNOTE_MCP_TOKEN`) gates every request** — there is no session; auth is checked per-request.
- **HTTPS only** — enforced by Vercel.
- **Token rotation** — change the env var, redeploy, update the connector URL in each Claude client.
- **Fail closed** — a deployment missing `ZKPNOTE_MCP_TOKEN` returns HTTP 500 rather than allowing anonymous access.
- Note: per-user authentication and rate limiting on `/api/mcp` are roadmap items; today the endpoint is a single-tenant shared-secret model.

## Troubleshooting

### "No notes found" but I have notes in the web app

If you created your vault with **Phantom wallet**, the MCP server (which uses a seed phrase) derives different encryption keys. Notes encrypted by Phantom can't be decrypted by the seed phrase, and vice versa. You need to use the **same seed phrase** you used to create the vault.

### Server doesn't appear in Claude

1. Check the path in your config — it must be an **absolute path** to `build/index.js`
2. Make sure you ran `npm run build` (the `build/` directory must exist)
3. Restart your Claude client after editing the config
4. Check Claude's MCP server logs for error messages

### "Failed to decrypt" errors

Your seed phrase doesn't match the vault's encryption key. Double-check you're using the exact 12-word phrase from when you created the vault.

### Tools are slow

Each vault operation does a full pull → decrypt → modify → encrypt → push cycle. Batch operations (`save_notes`) are significantly faster than individual `save_note` calls for multiple notes.

## Architecture

```
┌─────────────┐     stdio/SSE      ┌──────────────────┐     HTTPS     ┌──────────────┐
│ Claude /     │ ◄────────────────► │  ZKPnote MCP     │ ◄──────────► │  ZKPnote API │
│ AI Agent     │                    │  Server (local)   │              │  (Supabase)  │
└─────────────┘                    │                    │              └──────────────┘
                                   │  - Key derivation  │
                                   │  - E2E encryption  │              ┌──────────────┐
                                   │  - Note CRUD       │ ◄──────────► │  Solana RPC  │
                                   │  - Proof verify    │   (devnet)   │  (on-chain)  │
                                   └──────────────────┘              └──────────────┘
```

The MCP server is a local process that bridges your AI agent and your encrypted vault. It handles all cryptographic operations locally — the ZKPnote API only ever sees encrypted blobs.
