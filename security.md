# Security Model

ZKPnote is designed with a zero-knowledge architecture. The server and blockchain never have access to plaintext note content.

## Encryption

### Algorithm
- **Cipher:** XChaCha20-Poly1305 (via libsodium)
- **Key Derivation:** HKDF from BIP-39 seed or Phantom wallet signature
- **Key Size:** 256-bit
- **Nonce:** 24-byte random nonce per encryption operation

### What Is Encrypted
- Note title and content (individually encrypted)
- Folder names
- User profile (username, email, bio, profile picture)

### What Is NOT Encrypted
- Note ID, folder ID (UUIDs, non-sensitive)
- Timestamps (createdAt, updatedAt)
- Folder hierarchy (parentId relationships)
- On-chain vault hash (SHA-256 of the encrypted data, not the plaintext)

## Key Management

### Seed Phrase Mode
- 12-word BIP-39 mnemonic generates a 64-byte seed
- First 32 bytes derive the encryption key via HKDF
- Last 32 bytes derive the Solana keypair
- The seed phrase is the single root of trust; losing it means losing access

### Phantom Mode
- User signs a deterministic message: `"ChainNotes-vault-encryption-key-v1"`
- Ed25519 signatures are deterministic, so the same wallet always produces the same 64-byte signature
- First 32 bytes of the signature derive the encryption key
- Last 32 bytes derive an auth keypair for silent API authentication
- The connected Phantom wallet handles on-chain transaction signing

### Key Storage
- Keys are held in memory only (React state)
- No keys are persisted to localStorage, sessionStorage, or cookies
- Locking the vault clears all key material from memory
- Page refresh requires re-entering the seed phrase or reconnecting Phantom

## On-Chain Security

### Access Control
- **VaultAccount:** PDA seeded with `["vault", owner]` + `has_one = owner` constraint. Only the vault owner can update their vault hash.
- **ProgramConfig:** PDA seeded with `["config"]` + `has_one = authority`. Only the authority (deployer) can update treasury/fees.
- **ExecuteSale:** Treasury account validated on-chain against config via `constraint = treasury.key() == config.treasury`.

### Arithmetic Safety
- Fee calculation uses `checked_mul` and `checked_sub` to prevent overflow/underflow
- Price must be greater than zero (enforced by `require!` macro)

### Transaction Atomicity
- `execute_sale` performs two CPI transfers in a single transaction
- If either transfer fails, the entire transaction reverts
- No partial payment states are possible

## API Authentication

- API requests are authenticated using Ed25519 signatures from the auth keypair
- The server verifies the signature against the claimed public key
- In Phantom mode, the auth keypair is derived from the wallet signature, enabling silent API calls without Phantom popups

## Content Protection

### Similarity Detection
- New marketplace listings are compared against all existing listings using word-trigram Jaccard similarity
- Listings with >= 70% similarity to another seller's content are rejected
- Sellers can re-list their own content (same seller address is excluded from comparison)

### Purchase Tagging
- Notes acquired from the marketplace are tagged with the source listing ID
- Tagged notes are blocked from being listed for sale at both the client and server level

## Threat Model

| Threat | Mitigation |
|--------|-----------|
| Server compromise | Server only stores encrypted data; no plaintext access |
| Man-in-the-middle | All encryption happens client-side before transmission |
| Unauthorized vault access | Requires seed phrase or Phantom wallet signature |
| Content theft via marketplace | Similarity detection + purchase tagging |
| Treasury theft | Treasury address validated on-chain; only authority can change it |
| Fee manipulation | Fee basis points stored on-chain; only authority can update |
| Replay attacks | API auth uses fresh signatures with timestamp validation |

## Limitations

- **Browser memory:** Keys exist in JavaScript memory during a session, which is subject to browser-level attacks (malicious extensions, XSS)
- **Similarity detection:** Trigram-based comparison may not catch content that has been substantially rewritten
- **Off-chain bids:** Auction bids are not escrowed on-chain; the winner pays at settlement
- **No key rotation:** Changing the seed phrase creates a new vault; migrating an existing vault to a new key is not currently supported
