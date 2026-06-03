# 03 - Overlay topics & REST hydration

**Status: IMPLEMENTED (single overlay)** - overlay.peck.to. Federation/sync between overlays is ROADMAP ([`04-federation-roadmap.md`](04-federation-roadmap.md)).

Applications read from the overlay, not from chain directly. The overlay runs BRC-22 topic managers that admit outputs and lookup services that hydrate REST queries.

## 1. Topic managers (live)

| Topic | Admits | Lookup |
|---|---|---|
| `tm_peck` | Bitcoin Schema OP_RETURN content (posts, likes, follows, ...) | feed / search / thread |
| `tm_peck-bio-handle` | HandleToken | `ls_peck-bio-handle` |
| `tm_peck-bio-profile` | ProfileToken | `ls_peck-bio-profile` |

A lookup service: parses the layers, upserts a row keyed by outpoint, recomputes canonicality (canonical rule in [`01-token-state.md`](01-token-state.md)), persists Atomic-BEEF alongside the row for self-verifying hydration.

## 2. Topic-rooted state (normative)

Each topic maintains a state representation that is committed as a Merkle root, so topic-state-at-block-N is a verifiable commitment, not just a cache. Today a single overlay serves this; the inter-overlay sync and on-chain anchoring of these roots is roadmap.

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
