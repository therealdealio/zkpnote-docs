# ZKPnote MCP Server

The ZKPnote MCP (Model Context Protocol) server lets AI agents like Claude read, write, search, and verify notes in your encrypted vault. Your seed phrase stays local — all encryption happens on your machine before anything touches the network.

**Live deployment:** [zkpnote.vercel.app](https://zkpnote.vercel.app) (Solana devnet)

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
- **A ZKPnote account** — Create one at [zkpnote.vercel.app](https://zkpnote.vercel.app) using a seed phrase (not Phantom)
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
        "ZKPNOTE_API_URL": "https://zkpnote.vercel.app"
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
        "ZKPNOTE_API_URL": "https://zkpnote.vercel.app"
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
        "ZKPNOTE_API_URL": "https://zkpnote.vercel.app"
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
| `ZKPNOTE_API_URL` | No | `https://zkpnote.com` | ZKPnote API URL (use `https://zkpnote.vercel.app` for devnet) |

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

## Security Model

- **Seed phrase never leaves your machine.** It's used only by the local MCP server process to derive encryption keys.
- **All note content is encrypted with XChaCha20-Poly1305** before being sent to the ZKPnote API.
- **The server communicates over HTTPS** with the ZKPnote backend.
- **No data is written to disk** by the MCP server — everything runs in memory.
- **The API URL is the only external endpoint** the server contacts.

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
