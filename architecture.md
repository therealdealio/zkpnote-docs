# Architecture Overview

ZKPnote follows a privacy-first architecture where all sensitive operations happen client-side. The server never has access to plaintext note content.

## System Diagram

```
User Device                          Server                    Solana
+-----------------+                +-----------+            +-----------+
| Seed Phrase     |                |           |            |           |
|   |             |                | Encrypted |            | Vault PDA |
|   v             |                | Vault     |            | (hash +   |
| Key Derivation  |    encrypted   | Storage   |            |  owner +  |
|   |         |   |  ------------> |           |            |  count)   |
|   v         v   |                +-----------+            |           |
| Enc Key   Wallet|                                         | Config    |
|   |         |   |      signed tx                          | PDA      |
|   v         |   |  ---------------------------------------->         |
| Encrypt/    |   |                                         | (treasury |
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

## Data Flow

### Writing a Note
1. User types in the markdown editor
2. Note is encrypted client-side with XChaCha20-Poly1305
3. Encrypted note is stored in IndexedDB
4. Auto-sync pushes encrypted data to cloud storage
5. SHA-256 hash of the vault is written on-chain (Solana)

### Reading a Note
1. Pull encrypted vault from cloud storage
2. Decrypt client-side with the encryption key derived from seed phrase
3. Store decrypted notes in IndexedDB for fast access
4. Display in the markdown editor

### Marketplace Purchase
1. Buyer clicks "Buy Now" on a listing
2. `execute_sale` instruction is called on-chain
3. Solana program atomically splits payment: 98% to seller, 2% to treasury
4. Server records the purchase and releases the encrypted content
5. Content is decrypted and added to the buyer's vault

## On-Chain Accounts

| Account | Seeds | Description |
|---------|-------|-------------|
| VaultAccount | `["vault", owner]` | Stores vault hash, note count, timestamp |
| ProgramConfig | `["config"]` | Treasury address, fee basis points, authority |

## Folder Structure (Application)

```
src/
  app/              # Next.js App Router pages
    marketplace/    # Marketplace browse + analytics
    shared/         # Shared note viewer
    api/            # Server-side API routes
  components/       # React components
  contexts/         # VaultContext (global state)
  lib/              # Core libraries
    crypto.ts       # Encryption/decryption (libsodium)
    keyManager.ts   # Key derivation, wallet abstractions
    solana.ts       # Anchor program client
    sync.ts         # Cloud sync (push/pull)
    storage.ts      # IndexedDB operations
    auth.ts         # API authentication
  types/            # TypeScript interfaces

onchain/
  programs/
    chainnotes_vault/
      src/lib.rs    # Anchor smart contract
```
