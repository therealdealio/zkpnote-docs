# Marketplace API Reference

All marketplace operations use `POST /api/marketplace` with an `action` field in the JSON body.

## Actions

### `create` — List a Note for Sale

Create a new marketplace listing.

**Request:**
```json
{
  "action": "create",
  "noteTitle": "My Guide",
  "noteContent": "# Full markdown content...",
  "description": "A brief description visible to buyers",
  "sellerAddress": "...",
  "sellerUsername": "Alice",
  "sellerProfilePicture": null,
  "listingType": "copy",
  "price": 0.5,
  "category": "Guide",
  "previewMode": "masked",
  "previewChars": 200,
  "auctionDuration": 24,
  "reservePrice": 1.0
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `listingType` | string | Yes | `"copy"`, `"original"`, or `"auction"` |
| `previewMode` | string | No | `"masked"` (default), `"partial"`, or `"full"` |
| `previewChars` | number | No | Characters to show in partial mode (50-2000) |
| `auctionDuration` | number | No | Auction duration in hours (auction only) |
| `reservePrice` | number | No | Minimum price to sell (auction only) |

**Similarity Check:** Before creating, the API compares content against all existing listings from other sellers using trigram-based Jaccard similarity. Listings with 70%+ similarity to existing content are rejected.

**Response:**
```json
{ "listingId": "abc123..." }
```

### `browse` — List Marketplace Listings

Fetch all active listings with optional filters.

**Request:**
```json
{
  "action": "browse",
  "category": "Tutorial",
  "search": "solana",
  "seller": "..."
}
```

All filter fields are optional. Returns public info only (no `noteContent`).

**Response:**
```json
{
  "listings": [
    {
      "id": "...",
      "noteTitle": "...",
      "notePreview": "...",
      "maskedStructure": ["# ████ ████", "██ ██ ██ ██"],
      "previewMode": "masked",
      "previewText": null,
      "sellerAddress": "...",
      "sellerUsername": "...",
      "listingType": "copy",
      "price": 0.5,
      "category": "Tutorial",
      "salesCount": 3,
      "createdAt": 1712345678000,
      "auctionEndTime": null,
      "highestBid": null,
      "bids": []
    }
  ]
}
```

### `get` — Get Single Listing

Fetch a listing's details. If the buyer has purchased it, includes the full content.

**Request:**
```json
{
  "action": "get",
  "listingId": "abc123...",
  "buyerAddress": "..."
}
```

**Response:**
```json
{
  "listing": {
    "...all listing fields...",
    "noteContent": "# Full content (only if purchased)",
    "purchased": true
  }
}
```

### `purchase` — Record a Purchase

Called after the on-chain payment is confirmed.

**Request:**
```json
{
  "action": "purchase",
  "listingId": "abc123...",
  "buyerAddress": "...",
  "txSignature": "..."
}
```

**Validations:**
- Listing must exist and not be a sold original
- Buyer cannot purchase their own listing

**Response:**
```json
{
  "purchase": { "id": "...", "listingId": "...", "buyerAddress": "...", "txSignature": "...", "purchasedAt": 1712345678000 },
  "noteContent": "# Full markdown content",
  "noteTitle": "My Guide"
}
```

### `bid` — Place an Auction Bid

**Request:**
```json
{
  "action": "bid",
  "listingId": "abc123...",
  "bidderAddress": "...",
  "bidderUsername": "Bob",
  "amount": 0.75
}
```

**Validations:**
- Must be an active auction (not ended, not settled)
- Bid must exceed current highest bid by at least 0.001 SOL
- Cannot bid on your own auction

**Response:**
```json
{ "bid": { "bidder": "...", "amount": 0.75, "timestamp": 1712345678000 }, "highestBid": 0.75 }
```

### `settle` — Settle an Ended Auction

Called by the winning bidder after the auction ends and on-chain payment is confirmed.

**Request:**
```json
{
  "action": "settle",
  "listingId": "abc123...",
  "buyerAddress": "...",
  "txSignature": "..."
}
```

**Validations:**
- Auction must have ended
- Caller must be the highest bidder
- Reserve price must be met (if set)

**Response:** Same as `purchase`.

### `analytics` — Transaction Analytics

Returns all purchase transactions and aggregate statistics.

**Request:**
```json
{ "action": "analytics" }
```

**Response:**
```json
{
  "transactions": [...],
  "stats": {
    "totalSales": 15,
    "totalVolume": 7.5,
    "totalFees": 0.15,
    "uniqueBuyers": 8,
    "uniqueSellers": 5,
    "activeListings": 12
  }
}
```

### `delete` — Delete a Listing

Seller-only. Removes a listing from the marketplace.

**Request:**
```json
{
  "action": "delete",
  "listingId": "abc123...",
  "sellerAddress": "..."
}
```

## Content Protection

### Similarity Detection

When a new listing is created, its content is compared against all existing listings from other sellers:

1. Text is normalized (lowercase, markdown syntax stripped, whitespace collapsed)
2. Word trigrams are generated from both texts
3. Jaccard similarity coefficient is computed: `|A intersection B| / |A union B|`
4. If similarity >= 70%, the listing is rejected

This prevents buyers from reselling purchased content, even if they copy-paste it into a new note with minor edits.

### Purchase Tagging

Notes acquired through marketplace purchases are tagged with `purchasedFrom` (the listing ID). Tagged notes cannot be listed for sale — the sell button is disabled in the UI, and the API rejects the request.
