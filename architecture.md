# Architecture Overview

ZKPnote follows a privacy-first architecture where all sensitive operations happen client-side. The server never has access to plaintext note content.

**Live deployment:** [zkpnote.vercel.app](https://zkpnote.vercel.app)

## System Diagram

```
User Device                          Server                    Solana
+-----------------+                +-----------+            +-----------+
| Seed Phrase     |                |           |            |           |
|   |             |                | Supabase  |            | NoteProof |
|   v             |                | (vaults,  |            | PDA       |
| Key Derivation  |    encrypted   | listings, |            | (hash +   |
|   |         |   |  ------------> | purchases,|            |  owner +  |
|   v         v   |                | bids,     |            |  created) |
| Enc Key   Wallet|                | shares,   |            |           |
|   |         |   |      signed tx | proofs)   |            | Config    |
|   v         |   |  ----------------------------------------> PDA      |
| Encrypt/    |   |                +-----------+            | (treasury |
| Decrypt     |   |                                         |  + fee)  |
| Notes       v   |                                         |          |
|          Sign   |                                         | execute  |
|          Txs    |                                         | _sale    |
+-----------------+                                         +-----------+
```

## Key Derivation

Both login methods produce the **same Solana address, encryption key, and auth keypair** for the same wallet — so users can switch between seed phrase and Phantom and always access the same vault.

### Seed Phrase Mode
```
BIP-39 Mnemonic (12 words)
        |
        v
    64-byte seed
        |
  BIP-44 derivation (m/44'/501'/0'/0')
        |
        v
  Solana Keypair (same address as Phantom)
        |
  sign("ChainNotes-vault-encryption-key-v1")
        |
        v
  64-byte Ed25519 signature (deterministic)
    /          \
First 32B     Last 32B
    |              |
    v              v
Encryption    Auth Keypair
Key           (silent API auth)
```

### Phantom Mode
```
Phantom Wallet
        |
  sign("ChainNotes-vault-encryption-key-v1")
        |
        v
  64-byte Ed25519 signature (deterministic)
    /          \
First 32B     Last 32B
    |              |
    v              v
Encryption    Auth Keypair
Key           (silent API auth)
```

The Solana keypair is derived using BIP-44 (`m/44'/501'/0'/0'`) via `ed25519-hd-key`, matching Phantom and all standard Solana wallets. Both modes then sign the same deterministic message to derive identical encryption and auth keys. In Phantom mode, the connected wallet handles on-chain transaction signing; in seed phrase mode, the BIP-44 keypair signs directly.

### Phantom Signature Caching

To avoid requiring users to approve a Phantom signature popup on every page load, the wallet signature is cached in `sessionStorage`. This provides session persistence — the signature survives page refreshes within the same tab but is automatically cleared when the tab or browser window is closed. No key material is persisted to `localStorage` or cookies.

## Data Flow

### Writing a Note
1. User types in the rich text editor (Tiptap) or markdown editor
2. Note is encrypted client-side with XChaCha20-Poly1305
3. Encrypted note is stored in IndexedDB
4. Auto-sync pushes encrypted data to Supabase

**Offline support:** Notes are saved to IndexedDB (browser local storage) immediately on every keystroke. Cloud sync to Supabase happens on explicit sync or auto-sync. This means the app works offline — notes persist locally until connectivity is available.

### Per-Note Proof Registration
1. User triggers proof registration for a note
2. Client computes SHA-256 hash of the note: `SHA-256(title + "\n" + content)`
3. `register_proof` instruction is called on-chain with the hash
4. Solana creates a NoteProof PDA at `["proof", note_hash]` recording the owner and timestamp
5. Proof metadata (wallet address, hash, title, content, tx signature) is stored in the Supabase `proofs` table for public verification and similarity search

### Reading a Note
1. Pull encrypted vault from Supabase
2. Decrypt client-side with the encryption key derived from seed phrase
3. Store decrypted notes in IndexedDB for fast access
4. Display in the rich text editor (Tiptap) or markdown editor

### Verifying a Proof
1. User visits the `/verify` page (public, no authentication required)
2. User enters text to verify
3. Client computes SHA-256 hash locally (content never leaves the browser)
4. Hash is looked up against the Supabase `proofs` table via the `/api/proof` endpoint
5. If a match is found, the proof details (owner, timestamp, tx signature) are displayed

### Note Reordering
1. User drags a note via the grip handle in the sidebar
2. Drop indicator shows the target position
3. On drop, all notes in the folder receive updated `sortOrder` values (index-based)
4. Updated notes are encrypted and saved to IndexedDB
5. Auto-sync pushes the new order to Supabase
6. Notes with `sortOrder: null` fall back to `updatedAt` sorting (backward-compatible)

### Pop-Out Floating Window
1. User clicks the pop-out button in the editor toolbar
2. App creates a Document Picture-in-Picture window (Chrome/Edge) or falls back to `window.open()` (Safari/Firefox)
3. The floating window renders its own title input, content textarea, and word count
4. Edits in the floating window are debounced and synced back to the main vault via `debouncedSave`
5. The floating window is cleaned up automatically when the user switches notes or closes the main app

### Proof Recovery
1. User provides a transaction signature from a previously registered proof
2. `/api/proof` `recover` action looks up the proof in the `proofs` table
3. Ownership is verified (wallet address must match the proof's owner)
4. Full note content is returned for the user to restore to their vault

### Marketplace Purchase
1. Buyer clicks "Buy Now" on a listing
2. `execute_sale` instruction is called on-chain
3. Solana program atomically splits payment: 98% to seller, 2% to treasury
4. Supabase records the purchase and releases the encrypted content
5. Content is decrypted and added to the buyer's vault

## On-Chain Accounts

| Account | Seeds | Description |
|---------|-------|-------------|
| NoteProof | `["proof", note_hash]` | Per-note proof: owner, hash, timestamp |
| ProgramConfig | `["config"]` | Treasury address, fee basis points, authority |

## Supabase Tables

| Table | Description |
|-------|-------------|
| `vaults` | Encrypted vault blobs keyed by owner public key |
| `listings` | Marketplace listings (price, seller, metadata, encrypted content) |
| `purchases` | Records of completed marketplace purchases |
| `bids` | Open and accepted bids on marketplace listings |
| `shares` | Shared note access grants between users |
| `proofs` | Per-note proof records for public verification and similarity search |

## Folder Structure (Application)

```
src/
  app/              # Next.js App Router pages
    marketplace/    # Marketplace browse + analytics
    shared/         # Shared note viewer
    verify/         # Public proof verification page (no auth required)
    api/            # Server-side API routes
      proof/        # Proof store, verify, and search endpoint
  components/       # React components
  contexts/         # VaultContext (global state)
  lib/              # Core libraries
    crypto.ts       # Encryption/decryption (libsodium)
    keyManager.ts   # Key derivation, wallet abstractions
    solana.ts       # Anchor program client
    sync.ts         # Supabase sync (push/pull)
    storage.ts      # IndexedDB operations
    auth.ts         # API authentication
  types/            # TypeScript interfaces

onchain/
  programs/
    zkpnote/
      src/lib.rs    # Anchor smart contract

packages/
  mcp-server/       # MCP server for AI agent integration
    src/index.ts    # 15 tools: vault CRUD, marketplace, proofs, verify, search
```

## MCP Server

ZKPnote includes a Model Context Protocol (MCP) server that enables AI assistants (e.g., Claude Desktop) to interact with a user's vault programmatically.

### Architecture
- **Transport:** stdio (local process, no network exposure)
- **SDK:** `@modelcontextprotocol/sdk`
- **Key derivation:** Same HKDF path as the main app — derives encryption and auth keys from `ZKPNOTE_SEED_PHRASE` env var
- **API target:** `https://zkpnote.com` by default (configurable via `ZKPNOTE_API_URL`)

### Available Tools (15)

**Vault**
| Tool | Description |
|------|-------------|
| `save_note` | Create a new encrypted note |
| `list_notes` | List all notes in the vault |
| `read_note` | Read and decrypt a specific note |
| `update_note` | Update an existing note |
| `delete_note` | Delete a note |
| `list_folders` | List all folders |
| `reorder_notes` | Set custom sort order for notes |

**Proofs**
| Tool | Description |
|------|-------------|
| `verify_content` | Check if content has been proved on-chain |
| `search_similar` | Search for similar proved content |
| `list_proofs` | List all on-chain proofs for the wallet |
| `recover_note` | Recover lost note content from a proof tx, optionally restore to vault |

**Marketplace**
| Tool | Description |
|------|-------------|
| `browse_marketplace` | Browse/search marketplace listings with filters |
| `get_listing` | Get details of a specific listing |
| `cancel_listing` | Cancel own listing (restores note for originals) |
| `marketplace_analytics` | Get sales stats and transaction history |

## Rate Limiting

All API routes are rate-limited using Upstash Redis (`@upstash/ratelimit`) for distributed enforcement across Vercel edge instances. In local development, falls back to an in-memory store.

## Solana RPC Proxy

The `/api/rpc` route proxies Solana RPC calls from the browser to prevent exposing the RPC endpoint directly. A whitelist restricts access to 11 permitted methods (e.g., `getBalance`, `getLatestBlockhash`, `sendTransaction`). All other methods return 403.
