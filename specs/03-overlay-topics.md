# 03 — Overlay topics, submission and read API

- Status: IMPLEMENTED (one overlay). Federation between overlays is ROADMAP ([04](04-federation-roadmap.md)).
- Revision: 2
- Date: 2026-10-26

> **In short.** Apps do not scan the blockchain themselves. They send their
> transactions to an overlay, a server that checks each output against a set
> of rules (a "topic"), keeps the ones that fit, and answers questions about
> them over plain HTTP. This spec lists the topics, says how to submit a
> transaction, and describes the two read interfaces: the older `/v1` and the
> versioned `/v2` read model, which hands back posts already joined with their
> author and counts.

## 1. Topics (normative)

A topic manager (BRC-22) decides which outputs of a submitted transaction
belong to its topic. A lookup service indexes the admitted outputs.

| Topic | Admits | Spec | In the state root |
|---|---|---|---|
| `tm_social-content` | content records | [00](00-content-bitcoin-schema.md) | yes |
| `tm_social-profile` | ProfileToken outputs | [01](01-token-state.md) | yes |
| `tm_social-handle` | HandleToken outputs | [01](01-token-state.md) | yes |
| `tm_key-binding` | key-bindings | [06 §1](06-identity-attestations.md) | yes |
| `tm_identity-profile` | light profiles | [06 §2](06-identity-attestations.md) | yes |
| `tm_identity-handle` | light handle claims | [06 §3](06-identity-attestations.md) | yes |
| `tm_identity-cert` | certificate publications | [06 §4](06-identity-attestations.md) | yes |
| `tm_social-friend` | friend / unfriend records | [07](07-social-friend.md) | yes |
| `tm_social-lens` | moderation lenses | [06 §5](06-identity-attestations.md) | yes |
| `tm_social-classification` | content classifications | [06 §6](06-identity-attestations.md) | no |
| `tm_overlay-anchor` | state-anchor beacons | [08](08-state-anchoring.md) | no |

Topic managers are app-neutral: they admit any app's records that match the
rules, whatever the `app` value.

**Deployment-specific topics (not part of this standard).** The reference
deployment also runs `peck-schema` (the old name of the `tm_social-content`
manager; same admission rule, its own state root) and `tm_peck-cat` /
`tm_peck-cat-funding` (the peck.cat agent-token contracts). They are app
infrastructure with one implementation and are not specified here.

## 2. Submission (normative)

```
POST /submit
content-type: application/octet-stream
x-topics: ["tm_social-content"]           # JSON array of topic names
body: the transaction as BEEF or Atomic BEEF (BRC-62 / BRC-95)
200:  { "<topic>": { "outputsToAdmit": [<vout>...], "coinsToRetain": [...] }, ... }
```

- The overlay checks the merkle proofs in the BEEF, asks each named topic's
  manager which outputs it admits, and indexes them.
- Writers MUST send `x-topics`. Without it the reference overlay uses
  `["peck-schema"]`.
- A transaction whose outputs belong to several topics (an identity created
  in one transaction, [06 §0](06-identity-attestations.md)) is submitted once
  with every topic named.
- A repeat submission of a known transaction answers with an empty result,
  which looks the same as a refusal.
- The overlay answers before it has finished indexing. A read straight after
  a submission can miss the record.

The reference overlay also has `POST /submit-tx`, which indexes a raw
transaction's `OP_RETURN` outputs without proofs and without any topic
manager. Records that arrive only that way are not part of any topic's state
and are not anchored. It is not part of this standard.

## 3. Topic state (normative)

Each topic's admitted, unspent outputs form a set. Its state root is a
SHA-256 over that set, defined in [08 §1](08-state-anchoring.md), and served
at `GET /v1/topic/:topic/root`; `GET /state` serves every topic's root with
its latest on-chain anchor. The root is a flat hash, not a Merkle tree: it
commits to the whole set and gives no per-output inclusion proof.

The content topic's set holds only outputs admitted through `/submit`. Posts
that the reference index learns about from the chain indexer are served by
the read API but are not in any topic's state.

## 4. Threading

A reply's parent and thread root are derived as in
[00 §4](00-content-bitcoin-schema.md). `GET /v1/thread/:txid` serves a
thread, `GET /v1/thread-root/:txid` the root of a post's thread, and
`GET /v2/post/:txid` a post with its parent and every descendant reply.

## 5. `/v1` (informative)

The `/v1` routes are the reference deployment's original REST surface. They
stay up, but they are not a versioned contract and their shapes vary per
route. The ones other specs refer to:

```
GET  /v1/feed   /v1/post/:txid   /v1/thread/:txid   /v1/search?q=
GET  /v1/notifications/:address   /v1/friends/:address   /v1/lenses
POST /v1/identities/resolve   GET /v1/identities   GET /identity/:pubkey
GET  /resolve/:handle   /v1/bio/profile   /v1/bio/handle/:handle   /cert/verify/:sha
GET  /v1/topic/:topic/root   /state
```

## 6. `/v2`: the read model `peck-view/v1` (normative for consumers)

`/v2` serves a versioned, hydrated read model. Its contract is a JSON Schema
named `peck-view/v1`, owned by the overlay and vendored by the SDK
(`src/read/peck-view/peck-view.schema.json` in
[overlay-social/sdk](https://github.com/overlay-social/sdk)).

```
GET  /v2/feed                 FeedPage      (items, next)
GET  /v2/post/:txid           ThreadView    (post, parent, replies)
GET  /v2/post/:txid/reactions ReactionPage
POST /v2/posts                PostBatch
GET  /v2/profile/:key         ProfileView
POST /v2/viewer/state         ViewerState
GET  /v2/search  /v2/apps  /v2/stats  /v2/channels  /v2/identities
GET  /v2/lenses  /v2/authors  /v2/messages  /v2/payments  /v2/notifications
GET  /v2/block/:height  /v2/media/videos  /v2/media/videos/:key
```

Contract rules:

- Every view is the same for every viewer and can be cached by URL. Facts
  about one viewer (liked, following, blocked) come only from
  `POST /v2/viewer/state`.
- Within `peck-view/v1` the producer may add optional properties and add
  values to enums documented as open. It never removes, renames or retypes a
  property. Consumers MUST ignore properties they do not know. A breaking
  change is a new contract version served side by side.
- Paging is by keyset. A page's `next` is an opaque cursor; the client sends
  it back unchanged, with the same filters, as `cursor`.
- A `PostView` carries the author already resolved ([02](02-identity-projection.md))
  as an `AuthorView`, plus `type`, `kind`, `text`, `media`, `tags`, `channel`,
  `geo`, `parentTxid`, `threadRootTxid`, `refTxid` / `refKind`, `counts`,
  and `provenance` (see [00 §8](00-content-bitcoin-schema.md) for what the
  provenance grade does and does not prove).
- `text` is MAP `content`, else the B body, with inline base64 media moved
  to `media` and the result cut at a size limit (`textTruncated`). It is a
  rendering of the record, not its bytes.

`/v2/feed` filters include `app`, `type`/`types`, `kind`/`kinds`, `tag`,
`channel`, `author`, `since`, `until`, `rank` (`latest`, `top`,
`discussed`), `lens`, `has_geo`, `bbox`, `near` and `media`. The reference
deployment also applies its own operator lens to every feed.
