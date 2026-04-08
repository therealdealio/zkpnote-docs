# ZKPnote Developer Documentation

Technical documentation for developers building on or integrating with ZKPnote.

## Contents

- [Architecture Overview](./architecture.md)
- [Smart Contract Reference](./smart-contract.md)
- [Marketplace API](./marketplace-api.md)
- [Security Model](./security.md)

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 16, React 19, TypeScript, Tailwind CSS |
| Encryption | libsodium (XChaCha20-Poly1305), AES-256 key derivation |
| Blockchain | Solana, Anchor Framework 0.31.1 |
| Wallet | BIP-39 seed phrase + Phantom browser extension |
| Storage | IndexedDB (client), encrypted cloud sync (server) |
| Markdown | react-markdown with remark-gfm, rehype-highlight |

## Program ID

```
9AbLiwQ82manor3YyArrQhhpxPCFha5xbF187EtdDae5
```

## License

ZKPnote is proprietary software. This documentation is provided for integration and educational purposes.
