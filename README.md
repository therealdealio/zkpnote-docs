# ZKPnote Developer Documentation

Technical documentation for developers building on or integrating with ZKPnote.

**Live deployment:** [zkpnote.vercel.app](https://zkpnote.vercel.app)

## Contents

- [Architecture Overview](./architecture.md)
- [Smart Contract Reference](./smart-contract.md)
- [Marketplace API](./marketplace-api.md)
- [MCP Server Setup](./mcp-server.md)
- [Security Model](./security.md)

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 16, React 19, TypeScript, Tailwind CSS |
| Encryption | libsodium (XChaCha20-Poly1305), AES-256 key derivation |
| Blockchain | Solana, Anchor Framework 0.31.1 |
| Wallet | BIP-39 seed phrase + Phantom browser extension |
| Storage | IndexedDB (client), Supabase (server) |
| Editor | Tiptap (rich text), react-markdown with remark-gfm, rehype-highlight |

## Program ID

```
Ad67RwgTaeh77UQ5oZXAwt3fTvg3u5oNxNfcc3tGJLbc
```

(Solana devnet — program name: `zkpnote`)

## License

ZKPnote is proprietary software. This documentation is provided for integration and educational purposes.
