# 03 - Overlay topics & REST hydration

**Status: IMPLEMENTED (single overlay)** - overlay.peck.to. Federation/sync between overlays is ROADMAP ([`04-federation-roadmap.md`](04-federation-roadmap.md)).

Applications read from the overlay, not from chain directly. The overlay runs BRC-22 topic managers that admit outputs and lookup services that hydrate REST queries.

## 1. Topic managers (live)

| Topic | Admits | Spec |
|---|---|---|
| `tm_social-content` | Bitcoin Schema OP_RETURN content (posts, likes, follows, ...) | [`00`](00-content-bitcoin-schema.md) |
| `tm_social-profile` | ProfileToken (token-state) | [`01`](01-token-state.md) |
| `tm_social-handle` | HandleToken (token-state) | [`01`](01-token-state.md) |
| `tm_social-friend` | friend/unfriend attestations (mutual consent) | [`07`](07-social-friend.md) |
| `tm_key-binding` | posting-key ↔ identity-root bindings | [`06`](06-identity-attestations.md) |
| `tm_identity-profile` | light profile attestations | [`06`](06-identity-attestations.md) |
| `tm_identity-handle` | light handle claims (first-come) | [`06`](06-identity-attestations.md) |
| `tm_identity-cert` | BRC-52 cert publications | [`06`](06-identity-attestations.md) |
| `tm_overlay-anchor` | state-root anchor ordinals | [`08`](08-state-anchoring.md) |

A lookup service: parses the layers, upserts a row keyed by outpoint, recomputes canonicality (canonical rule in [`01-token-state.md`](01-token-state.md)), persists Atomic-BEEF alongside the row for self-verifying hydration.

## 2. Topic-rooted state (normative)

Each topic maintains a state representation committed as a Merkle root, so topic-state-at-block-N is a verifiable commitment, not just a cache. The roots are **anchored on-chain** and auditable — see [`08-state-anchoring.md`](08-state-anchoring.md). Inter-overlay sync remains roadmap ([`04`](04-federation-roadmap.md)).

## 3. REST hydration (live)

```
GET /v1/feed?app=&limit=&offset=
GET /v1/post/:txid  /v1/thread/:txid  /v1/search?q=
POST /v1/identities/resolve               -> see 02
GET /identity/:pubkey  /resolve/:handle   -> see 02
GET /v1/bio/profile  /v1/bio/handle/:handle  /cert/verify/:sha
```

Pagination uses a structured cursor (`before_height` + `before_outpoint` tiebreaker), never offset, for token queries.

## 4. Submission

Clients submit transactions to the overlay asynchronously (peck-broadcaster pattern). Idempotent submission (dedup by content-hash/nonce) is partially present and formalized in the federation roadmap.
