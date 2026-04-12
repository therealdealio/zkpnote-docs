# Smart Contract Reference

ZKPnote uses an Anchor-based Solana program (`zkpnote`) for per-note proof registration and marketplace payments.

**Program ID:** `Ad67RwgTaeh77UQ5oZXAwt3fTvg3u5oNxNfcc3tGJLbc`
**Anchor Version:** 0.31.1
**Solana Version:** 2.x

## Instructions

The program has 6 instructions: `register_proof`, `initialize_config`, `update_config`, `record_share_view`, `sign_nda`, and `execute_sale`.

### `register_proof`

Registers a per-note proof on-chain. The proof PDA is derived from the note's SHA-256 hash, establishing first-to-register ownership of the content.

| Parameter | Type | Description |
|-----------|------|-------------|
| `note_hash` | `[u8; 32]` | SHA-256 hash of the note (title + "\n" + content) |

**Accounts:**
| Account | Type | Description |
|---------|------|-------------|
| `proof` | PDA `["proof", note_hash]` | NoteProof account (init) |
| `owner` | Signer, mut | Wallet registering the proof |
| `system_program` | Program | System program |

**Errors:**
- `ZkpnoteError::InvalidHash` — hash must be exactly 32 bytes

### `initialize_config`

One-time initialization of the marketplace configuration. The caller becomes the authority (admin).

| Parameter | Type | Description |
|-----------|------|-------------|
| `treasury` | `Pubkey` | Wallet that receives marketplace fees |
| `fee_basis_points` | `u16` | Fee in basis points (200 = 2%) |

**Accounts:**
| Account | Type | Description |
|---------|------|-------------|
| `config` | PDA `["config"]` | Config account (init) |
| `authority` | Signer, mut | Admin wallet |
| `system_program` | Program | System program |

### `update_config`

Updates the treasury address and/or fee. Authority only.

| Parameter | Type | Description |
|-----------|------|-------------|
| `treasury` | `Pubkey` | New treasury address |
| `fee_basis_points` | `u16` | New fee in basis points |

**Accounts:**
| Account | Type | Description |
|---------|------|-------------|
| `config` | PDA `["config"]` | Config account (mut, has_one = authority) |
| `authority` | Signer | Admin wallet |

### `record_share_view`

Records a reader's agreement to view a shared note. The reader signs and pays rent. The note must already have an on-chain proof.

| Parameter | Type | Description |
|-----------|------|-------------|
| `share_id` | `[u8; 16]` | First 16 bytes of the share UUID |
| `note_hash` | `[u8; 32]` | SHA-256 hash of the note (links to the existing NoteProof PDA) |

**Accounts:**
| Account | Type | Description |
|---------|------|-------------|
| `agreement` | PDA `["share", share_id, reader]` | ShareAgreement account (init) |
| `proof` | PDA `["proof", note_hash]` | Existing NoteProof (read-only) |
| `reader` | Signer, mut | Reader wallet (pays rent) |
| `system_program` | Program | System program |

### `sign_nda`

Signs an NDA contract for a shared note. The note owner signs and pays rent. `has_one = owner` on the proof PDA ensures only the note author can approve the NDA. The reader's full name is hashed client-side before submission.

| Parameter | Type | Description |
|-----------|------|-------------|
| `share_id` | `[u8; 16]` | First 16 bytes of the share UUID |
| `note_hash` | `[u8; 32]` | SHA-256 hash of the note |
| `name_hash` | `[u8; 32]` | SHA-256 hash of the reader's typed name |

**Accounts:**
| Account | Type | Description |
|---------|------|-------------|
| `nda_contract` | PDA `["nda", share_id, reader]` | NdaContract account (init) |
| `proof` | PDA `["proof", note_hash]` | Existing NoteProof (`has_one = owner`) |
| `owner` | Signer, mut | Note owner (pays rent) |
| `reader` | AccountInfo | Reader wallet (not a signer) |
| `system_program` | Program | System program |

### `execute_sale`

Executes a marketplace sale with atomic payment splitting. Sends (100% - fee) to seller and fee to treasury in a single transaction.

| Parameter | Type | Description |
|-----------|------|-------------|
| `price_lamports` | `u64` | Total price in lamports |

**Accounts:**
| Account | Type | Description |
|---------|------|-------------|
| `config` | PDA `["config"]` | Config account (read-only) |
| `buyer` | Signer, mut | Buyer wallet |
| `seller` | AccountInfo, mut | Seller wallet |
| `treasury` | AccountInfo, mut | Treasury (must match config.treasury) |
| `system_program` | Program | System program |

**Fee Calculation:**
```
fee = price_lamports * fee_basis_points / 10000
seller_amount = price_lamports - fee
```

Uses checked arithmetic (`checked_mul`, `checked_sub`) for overflow protection.

## Account Structures

### NoteProof (73 bytes + 8 discriminator)

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `owner` | Pubkey | 32 | Wallet that registered this proof |
| `note_hash` | [u8; 32] | 32 | SHA-256 hash of the note content |
| `created_at` | i64 | 8 | Unix timestamp of proof registration |
| `bump` | u8 | 1 | PDA bump seed |

**PDA Seeds:** `[b"proof", note_hash.as_ref()]`

### ProgramConfig (67 bytes + 8 discriminator)

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `authority` | Pubkey | 32 | Admin wallet |
| `treasury` | Pubkey | 32 | Fee recipient wallet |
| `fee_basis_points` | u16 | 2 | Fee percentage (200 = 2%) |
| `bump` | u8 | 1 | PDA bump seed |

### ShareAgreement (121 bytes + 8 discriminator)

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `owner` | Pubkey | 32 | Note owner who created the share |
| `reader` | Pubkey | 32 | Wallet that viewed the note |
| `note_hash` | [u8; 32] | 32 | SHA-256 hash linking to the NoteProof |
| `share_id` | [u8; 16] | 16 | First 16 bytes of the share UUID |
| `created_at` | i64 | 8 | Unix timestamp when the view was recorded |
| `bump` | u8 | 1 | PDA bump seed |

**PDA Seeds:** `[b"share", share_id, reader.as_ref()]`

### NdaContract (153 bytes + 8 discriminator)

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `owner` | Pubkey | 32 | Note owner who approved the NDA |
| `reader` | Pubkey | 32 | Reader wallet |
| `note_hash` | [u8; 32] | 32 | SHA-256 hash linking to the NoteProof |
| `share_id` | [u8; 16] | 16 | First 16 bytes of the share UUID |
| `name_hash` | [u8; 32] | 32 | SHA-256 hash of the reader's typed name |
| `created_at` | i64 | 8 | Unix timestamp when the NDA was signed |
| `bump` | u8 | 1 | PDA bump seed |

**PDA Seeds:** `[b"nda", share_id, reader.as_ref()]`

## Error Codes

| Code | Name | Message |
|------|------|---------|
| 6000 | InvalidHash | Hash must be 32 bytes |
| 6001 | InvalidTreasury | Invalid treasury account |
| 6002 | InvalidPrice | Price must be greater than zero |

## Security Properties

- **PDA-based proof uniqueness:** NoteProof accounts are derived from `["proof", note_hash]`, ensuring exactly one proof can exist per unique hash. The first wallet to register a hash owns the proof.
- **Authority enforcement:** Config updates require the original authority signer via `has_one` constraint
- **Treasury validation:** `execute_sale` validates the treasury account matches the on-chain config via constraint
- **Atomic transfers:** Payment splitting happens in a single transaction; if either transfer fails, both revert
- **Overflow protection:** All fee math uses checked arithmetic
