# 05 — Additional write channels and token economics

- Status: ROADMAP, except §5 (the token/post bridge), which is PARTIAL.
- Revision: 2
- Date: 2026-10-26

> **In short.** Today a social record rides in a data output of its own
> ([00](00-content-bitcoin-schema.md)). This file collects designs for other
> ways to write: data carried inside a payment, posts that are owned and can
> be sold, paid access to content, and a faster feed from the network. Each
> section says how far it has come. Treat everything not marked otherwise as a
> plan.

## 1. Input-script data (ROADMAP)

Data carried in the unlocking script of an ordinary P2PKH spend as
`<chunk> OP_DROP` pairs before `<sig> <pubkey>`, so that a payment and its
record are one event: tip with a comment, paid follow, paywall receipt. The
AIP preimage rule is the same as for `OP_RETURN`
([00 §7](00-content-bitcoin-schema.md)): only data pushes count, and the
`OP_DROP`s do not. Indexers would have to scan input scripts, which most
Bitcoin Schema parsers do not.

## 2. PostToken family and marketplace (ROADMAP)

Posts as 1-satoshi tokens (PostToken, with app subtypes such as canvas, event
and channel tokens). Minting requires a canonical handle. First surface:
`update`, `transfer`, `burn`, `reply`; an edit burns the token and mints a
child, and the content reference never changes. Selling, bidding, royalties
and like-as-payment wait for real usage. No PostToken topic or contract runs
today.

## 3. Paid content (ROADMAP)

Per-post paywall: the content is encrypted and stored by content address
(UHRP, BRC-26), the token or record states a price, the reader pays the owner
once or through a payment channel, and the overlay releases the key on proof
of payment. The reference overlay contains a payment-channel paywall
implementation, but it is switched off in the reference deployment and no
content is sold through it.

## 4. Teranode direct consume (ROADMAP)

Move the indexer off third-party relays onto Teranode's own message topics
(validated transactions, subtrees, final blocks, invalidations). Lower
latency and no third-party dependency. Not a blocker: the current relay path
works.

## 5. Token/post bridge (PARTIAL)

A transaction that updates a token MAY also carry an ordinary content record
(`type post`), so that readers who know only [00](00-content-bitcoin-schema.md)
see a post and token-aware readers link the two.

Output order:

```
[0] token state output (1 sat)
[1] OP_RETURN token beacon (Layer B, 01)
[2] OP_RETURN content record (only when the update has post text)
[3] change
```

Implemented: the reference overlay links the post to the token for
ProfileToken, HandleToken and the deployment-specific peck.cat token, stores
the link with the post, and serves posts per token at
`GET /v1/token-posts/:txid.vout`. An update without a content record creates
no post. Not implemented: PostToken (§2), and a link visible in the `/v2`
read model.
