# Smart Contract Reference

ZKPnote uses an Anchor-based Solana program for on-chain vault verification and marketplace payments.

**Program ID:** `9AbLiwQ82manor3YyArrQhhpxPCFha5xbF187EtdDae5`
**Anchor Version:** 0.31.1
**Solana Version:** 2.x

## Instructions

### `initialize_vault`

Creates a new vault record on-chain for a user. Stores a SHA-256 hash of the encrypted vault data as a verification anchor.

| Parameter | Type | Description |
|-----------|------|-------------|
| `vault_hash` | `[u8; 32]` | SHA-256 hash of the encrypted vault |
| `note_count` | `u32` | Number of notes in the vault |

**Accounts:**
| Account | Type | Description |
|---------|------|-------------|
| `vault` | PDA `["vault", owner]` | Vault account (init) |
| `owner` | Signer, mut | Wallet creating the vault |
| `system_program` | Program | System program |

### `update_vault`

Updates the vault hash after a sync.

| Parameter | Type | Description |
|-----------|------|-------------|
| `vault_hash` | `[u8; 32]` | New SHA-256 hash |
| `note_count` | `u32` | Updated note count |

**Accounts:**
| Account | Type | Description |
|---------|------|-------------|
| `vault` | PDA `["vault", owner]` | Vault account (mut, has_one = owner) |
| `owner` | Signer | Vault owner |

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

### VaultAccount (81 bytes + 8 discriminator)

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `owner` | Pubkey | 32 | Wallet that owns this vault |
| `vault_hash` | [u8; 32] | 32 | SHA-256 hash of encrypted vault |
| `note_count` | u32 | 4 | Number of notes |
| `updated_at` | i64 | 8 | Unix timestamp of last update |
| `bump` | u8 | 1 | PDA bump seed |

### ProgramConfig (67 bytes + 8 discriminator)

| Field | Type | Size | Description |
|-------|------|------|-------------|
| `authority` | Pubkey | 32 | Admin wallet |
| `treasury` | Pubkey | 32 | Fee recipient wallet |
| `fee_basis_points` | u16 | 2 | Fee percentage (200 = 2%) |
| `bump` | u8 | 1 | PDA bump seed |

## Error Codes

| Code | Name | Message |
|------|------|---------|
| 6000 | InvalidTreasury | Invalid treasury account |
| 6001 | InvalidPrice | Price must be greater than zero |

## Security Properties

- **PDA-based access control:** Vault accounts are derived from `["vault", owner]`, ensuring only the owner can update their vault
- **Authority enforcement:** Config updates require the original authority signer via `has_one` constraint
- **Treasury validation:** `execute_sale` validates the treasury account matches the on-chain config via constraint
- **Atomic transfers:** Payment splitting happens in a single transaction; if either transfer fails, both revert
- **Overflow protection:** All fee math uses checked arithmetic
