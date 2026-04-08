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

### Seed Phrase Mode
```
BIP-39 Mnemonic (12 words)
        |
        v
    64-byte seed
    /          \
First 32B     Last 32B
    |              |
    v              v
Encryption    Solana Keypair
Key           (wallet + auth)
```

### Phantom Mode
```
Phantom Wallet
        |
  sign("ZKPnote-vault-encryption-key-v1")
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

In Phantom mode, the connected wallet is used for on-chain transactions, while the derived auth keypair handles API authentication without popup prompts.

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
```
