---
name: jito-mev-grpc
description: "Use when connecting to Jito gRPC or streaming Solana txs."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [jito, solana, mev, grpc, block-engine, yellowstone]
    related_skills: []
---

# Jito MEV gRPC + Solana Transaction Stream

Connect to Jito's Block Engine over gRPC (MEV: bundles, tip accounts, leaders) and stream raw Solana transactions via Yellowstone gRPC. Reference implementation at `~/jito-mev` (Node.js).

## When to Use

- User wants to install/use "Jito MEV software" and receive Solana transaction data over gRPC
- Submitting bundles, querying tip accounts / next leader / regions on Jito Block Engine
- Streaming live Solana transactions (mempool feed) for MEV bots

Don't use for: Jito's *public geyser tx feed* (`mainnet.rpc.jito.wtf:10000`) — it is **sunset** (DNS removed for most regions, port refused). Jito ShredStream also shut down 2026-09-05.

## Key Facts (verify before acting)

- **Jito gRPC that still works:** Block Engine `SearcherService` + `AuthService` at `mainnet.block-engine.jito.wtf:443` (TLS gRPC). Regional: `<region>.mainnet.block-engine.jito.wtf:443` where region ∈ `amsterdam, dublin, frankfurt, london, ny, slc, singapore, tokyo`.
- **Protos:** official `jito-labs/mev-protos` repo (branch `master`): `searcher.proto`, `auth.proto`, `bundle.proto`, `packet.proto`, `shared.proto`, `block_engine.proto`, `relayer.proto`, `block.proto`, `shredstream.proto`. Imports `google/protobuf/timestamp.proto` (fetch from `protocolbuffers/protobuf` `src/google/protobuf/timestamp.proto`).
- **Raw tx stream replacement:** Yellowstone gRPC (`rpcpool/yellowstone-grpc` → `yellowstone-grpc-proto/proto/geyser.proto` + `solana-storage.proto`). Providers: PublicNode `solana-yellowstone-grpc.publicnode.com:443` (free token at allnodes.com/publicnode), Helius `mainnet.helius-rpc.com:443`, Triton/QuickNode. Auth via `x-token` metadata header.
- **Rate limit:** Block Engine defaults to 1 req/sec per IP per region. Space unary calls ~1.2s apart. Auth raises it; requires pubkey registered with Jito (Discord).

## Prerequisites

- Node 18+ and `npm` (or any gRPC-capable stack; this skill is Node-flavored)
- Deps: `@grpc/grpc-js @grpc/proto-loader @solana/web3.js bs58 tweetnacl`
- Yellowstone token (free) if streaming transactions; Jito-registered pubkey if using auth

## How to Run

```bash
# scaffold + deps
mkdir jito-mev && cd jito-mev && npm init -y
npm install @grpc/grpc-js @grpc/proto-loader @solana/web3.js bs58 tweetnacl

# fetch protos into protos/ (mev-protos files + timestamp.proto + geyser.proto + solana-storage.proto)
# then run the reference demo (if ~/jito-mev exists)
node ~/jito-mev/index.js                                   # Block Engine read-only demo
YELLOWSTONE_TOKEN=<token> node ~/jito-mev/src/txstream.js  # tx stream
```

## Procedure

1. **Fetch protos** — download `jito-labs/mev-protos` `.proto` files plus `google/protobuf/timestamp.proto` into a `protos/` dir (keep the `google/protobuf/` subpath). Check imports with `search_files` (`^import`).
2. **Load with proto-loader** — options that work: `keepCase:false, longs:String, enums:String, defaults:false, oneofs:true, includeDirs:[PROTO_DIR]`. `longs:String` is required (Solana slots exceed 2^53).
3. **Dial TLS gRPC** — `new grpc.Client(host, grpc.credentials.createSsl())` to `mainnet.block-engine.jito.wtf:443`. The `SearcherService` is `pkg.searcher.SearcherService`.
4. **Call read-only methods** — `GetRegions`, `GetTipAccounts`, `GetNextScheduledLeader` need no auth. Space calls 1.2s apart or you hit `RESOURCE_EXHAUSTED: Rate limit exceeded`.
5. **Auth (optional)** — `AuthService.GenerateAuthChallenge({role:'SEARCHER', pubkey})` → sign `"<pubkeyB58>-<challenge>"` with ed25519 → `GenerateAuthTokens` → attach `authorization: Bearer <accessToken>`. Unregistered pubkey returns `PERMISSION_DENIED: The supplied pubkey is not authorized to generate a token` (expected).
6. **Stream bundle results** — `SubscribeBundleResults({})` returns a stream of `BundleResult` (accepted/rejected/processed/finalized/dropped). Opens cleanly even unauthenticated.
7. **Stream raw tx (Yellowstone)** — load `geyser.proto`, subscribe with `{ transactions: { client: { vote:false, failed:false } }, commitment:'PROCESSED' }` + `x-token` metadata. Each `update.transaction.transaction.signature` is a Solana tx signature (bs58).

## Pitfalls

- **`GetConnectedLeaders` fails with `Response message parsing error: Cannot read properties of null (reading 'slots')`** — a `@grpc/proto-loader` bug deserializing `map<string, SlotList>`, NOT a Jito/server error. The Rust client and JSON-RPC return it fine. Don't chase it in Node; use JSON-RPC `getConnectedLeaders` if needed.
- **`defaults:true` vs `false`** does not change field access; but `keepCase:false` (camelCase) is what makes `currentRegion`, `clientPubkey`, `accessToken` work. If you set `keepCase:true`, you must use snake_case field names.
- **Port 10000 is dead** — `tokyo.mainnet.rpc.jito.wtf` still resolves but refuses 10000; other regions don't resolve at all. Any "connect to mainnet.rpc.jito.wtf" recipe is stale.
- **Field names:** with camelCase, `GetRegions` → `currentRegion`/`availableRegions`; `GenerateAuthTokens` request → `clientPubkey`/`signedChallenge`; response → `accessToken`/`refreshToken`.

## Verification

- `GetTipAccounts` returns exactly the 8 known Jito tip accounts (`96gYZGLnJY...`, `HFqU5x63VT...`, etc.).
- `GetRegions` returns a non-empty `currentRegion` and 8 `availableRegions`.
- `GetNextScheduledLeader` returns a real `currentSlot` (4.5e8 range) and `nextLeaderIdentity`.
- Yellowstone `Subscribe` without a token returns `PERMISSION_DENIED: ... requires a personal token` — proves the connection + proto load work; with a valid token it streams `[tx] slot=... sig=...` lines.
