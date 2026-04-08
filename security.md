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
- On-chain proof hash (SHA-256 of the plaintext note, not the encryption key)

## Key Management

### Seed Phrase Mode
- 12-word BIP-39 mnemonic generates a 64-byte seed
- First 32 bytes derive the encryption key via HKDF
- Last 32 bytes derive the Solana keypair
- The seed phrase is the single root of trust; losing it means losing access

### Phantom Mode
- User signs a deterministic message: `"ZKPnote-vault-encryption-key-v1"`
- Ed25519 signatures are deterministic, so the same wallet always produces the same 64-byte signature
- First 32 bytes of the signature derive the encryption key
- Last 32 bytes derive an auth keypair for silent API authentication
- The connected Phantom wallet handles on-chain transaction signing

### Key Storage
- Keys are held in memory only (React state)
- No keys are persisted to localStorage or cookies
- Locking the vault clears all key material from memory
- Page refresh requires re-entering the seed phrase or reconnecting Phantom

### Phantom Signature Caching
- The Phantom wallet signature used for key derivation is cached in `sessionStorage`
- This avoids requiring a Phantom popup on every page refresh within the same session
- `sessionStorage` is scoped to the browser tab and is automatically cleared when the tab or browser window is closed
- This is the only value stored in `sessionStorage`; no encryption keys or private keys are cached

## On-Chain Security

### Access Control
- **NoteProof:** PDA seeded with `["proof", note_hash]`. The first wallet to register a given hash owns the proof. Once created, the proof is immutable.
- **ProgramConfig:** PDA seeded with `["config"]` + `has_one = authority`. Only the authority (deployer) can update treasury/fees.
- **ExecuteSale:** Treasury account validated on-chain against config via `constraint = treasury.key() == config.treasury`.

### Per-Note Proof System
- Each note can have an on-chain proof registered via `register_proof`
- The proof hash is `SHA-256(title + "\n" + content)`, computed client-side
- PDA derivation from the hash ensures exactly one proof can exist per unique content
- First-to-register wins: once a proof is created, no other wallet can claim the same hash
- This prevents content theft claims — the on-chain timestamp proves who registered the content first

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

## Proof Verification Privacy

- The `/verify` page is public and requires no authentication
- Hash computation happens entirely client-side — the plaintext content entered for verification never leaves the browser
- Only the computed hash is sent to the server for lookup
- This means a user can verify whether content has been registered without exposing the content to the server

## Content Protection

### Similarity Detection
- New marketplace listings are compared against all existing listings using word-trigram Jaccard similarity
- Listings with >= 70% similarity to another seller's content are rejected
- Sellers can re-list their own content (same seller address is excluded from comparison)

### Purchase Tagging
- Notes acquired from the marketplace are tagged with the source listing ID
- Tagged notes are blocked from being listed for sale at both the client and server level

### Per-Note Proof
- On-chain proofs provide cryptographic evidence of authorship with a Solana-timestamped record
- If a content theft dispute arises, the proof with the earlier `created_at` timestamp establishes priority
- Proofs are stored in the Supabase `proofs` table for fast lookup and similarity search via the `/api/proof` endpoint

## API Security

### Rate Limiting
All API routes (`/api/vault`, `/api/marketplace`, `/api/proof`, `/api/rpc`) are rate-limited using Upstash Redis (`@upstash/ratelimit`). This provides distributed rate limiting across Vercel edge instances — requests from the same IP are tracked globally, not per-instance. In local development, falls back to an in-memory store.

### RPC Proxy Whitelist
Browser-side Solana RPC calls are routed through `/api/rpc` to avoid exposing the RPC endpoint directly. The proxy enforces a strict whitelist of 11 permitted methods:
- `getBalance`, `getLatestBlockhash`, `getRecentBlockhash`, `sendTransaction`, `confirmTransaction`, `getTransaction`, `getSignatureStatuses`, `getAccountInfo`, `getMinimumBalanceForRentExemption`, `requestAirdrop`, `getMultipleAccounts`

Any method not on this list returns HTTP 403. This prevents abuse of the RPC endpoint for arbitrary Solana queries.

## Threat Model

| Threat | Mitigation |
|--------|-----------|
| Server compromise | Server only stores encrypted data; no plaintext access |
| Man-in-the-middle | All encryption happens client-side before transmission |
| Unauthorized vault access | Requires seed phrase or Phantom wallet signature |
| Content theft via marketplace | Similarity detection + purchase tagging |
| Content theft claims | Per-note on-chain proof with first-to-register timestamp |
| Treasury theft | Treasury address validated on-chain; only authority can change it |
| Fee manipulation | Fee basis points stored on-chain; only authority can update |
| Replay attacks | API auth uses fresh signatures with timestamp validation |
| Session hijacking via cached signature | Phantom signature cached in sessionStorage only; cleared on tab close |
| API abuse / DDoS | Distributed rate limiting via Upstash Redis across all edge instances |
| RPC endpoint abuse | Whitelist restricts Solana RPC proxy to 11 permitted methods; all others return 403 |

## Limitations

- **Browser memory:** Keys exist in JavaScript memory during a session, which is subject to browser-level attacks (malicious extensions, XSS)
- **Similarity detection:** Trigram-based comparison may not catch content that has been substantially rewritten
- **Off-chain bids:** Auction bids are not escrowed on-chain; the winner pays at settlement
- **No key rotation:** Changing the seed phrase creates a new vault; migrating an existing vault to a new key is not currently supported
- **Proof immutability:** Once a proof is registered on-chain, it cannot be updated or deleted. If content changes, a new proof must be registered with the updated hash.
