# overlay.social specs

Open specifications for social applications on BSV that keep the user's
data and identity in the user's own hands.

## What is overlay.social, in plain words

When you post on a normal social network, the company owns the database: your
posts, your name and who you follow live on its servers, and it decides what
others see. overlay.social is a set of rules for doing the same things
differently. Each post, profile or friendship is a small record the user signs
with their own wallet and writes to the BSV blockchain, where nobody can
change or remove it. Servers called overlays collect those records, check them
and serve them quickly to apps, but any app can read the same records, and
anyone can check an overlay's answers against the chain. You can change apps
without losing your name, your posts or your friends. These documents say
exactly how the records are written and read, so that independent apps and
overlays agree.

## Honesty contract

Every spec declares a status:

- **IMPLEMENTED**: code runs in production against it today.
- **PARTIAL**: some of it runs; the spec says which part.
- **ROADMAP**: a design we intend to build. Do not assume an endpoint exists
  because it is described.

Each spec starts with a header (`Status`, `Revision`, `Date`) and an "In
short" block in plain language. The "In short" block is an introduction, not
part of the normative text. In the normative text, MUST, SHOULD and MAY are
used as in RFC 2119 and RFC 8174. Terms are explained in
[`GLOSSARY.md`](GLOSSARY.md).

## Reading order

| # | Spec | Status |
|---|---|---|
| — | [Glossary](GLOSSARY.md) | — |
| 00 | [Content records (Bitcoin Schema)](specs/00-content-bitcoin-schema.md) | IMPLEMENTED |
| 03 | [Overlay topics, submission and read API](specs/03-overlay-topics.md) | IMPLEMENTED (one overlay) |
| 06 | [Self-signed records: identity, key-binding, certificates, lenses, classifications](specs/06-identity-attestations.md) | IMPLEMENTED; classifications PARTIAL |
| 02 | [Identity resolution](specs/02-identity-projection.md) | IMPLEMENTED |
| 07 | [Friend: mutual-consent relationships](specs/07-social-friend.md) | IMPLEMENTED |
| 08 | [State anchoring](specs/08-state-anchoring.md) | IMPLEMENTED; outpoint-set listing ROADMAP |
| 09 | [Documents](specs/09-documents.md) | PARTIAL |
| 01 | [Token-state identity (ProfileToken, HandleToken)](specs/01-token-state.md) | IMPLEMENTED |
| 05 | [Additional write channels and token economics](specs/05-write-channels-roadmap.md) | ROADMAP; token/post bridge PARTIAL |
| 04 | [Federation](specs/04-federation-roadmap.md) | ROADMAP |

Start with 00 (how a record is written) and 03 (how an overlay takes and
serves it). 06, 02 and 07 cover identity and relationships, 08 and 09 how
state and documents are pinned to the chain. 01, 05 and 04 are the older
token family and the roadmap.

## What runs today and what does not

Runs today (reference deployment):

- Content records with BRC77 or Bitcoin Signed Message signatures, replies,
  quotes, reposts, likes, follows, tips, channels, locations.
- Topic managers for content, both identity families, key-bindings,
  certificates, friendships, lenses and classifications.
- Identity resolution in one batch call, with bound keys collapsed.
- The `/v2` read model `peck-view/v1`.
- State roots for 12 topics, anchored on chain whenever they change.
- Document revisions as posts (`kind doc`), admitted, listed and anchored.

Not built yet:

- A read model for document history (09), and serving the outpoint sets
  behind state roots (08).
- Federation between overlays (04).
- Input-script data, PostToken, paid content, Teranode direct consume (05).

## Layer map

```
Application     any app (reference apps: peck.to, peck.bio, peck.press, peck.world)
Read            overlay: topic managers, /v1 and /v2 over HTTP      one overlay today; federation is roadmap
Write           OP_RETURN content records (live), token state (live), input-script data (roadmap)
Identity        BRC-100 wallets, BRC-42/43 key derivation, BRC-3/77 signatures, BRC-52 certificates
Substrate       BSV mainnet
```

## Reference deployment

The reference overlay is [overlay.peck.to](https://overlay.peck.to). Its
reference client library is [overlay-social/sdk](https://github.com/overlay-social/sdk).
Neither is the standard: another overlay that follows these specs serves the
same records.

## Open questions

- Light handle claims (06 §3) and friend withdrawals (07 §3) are ordered by
  admission on one overlay. Two overlays could order them differently. A
  chain-derived order (block height and position, as in 09 §4.2) would make
  them agree.
- Key-binding precedence uses the writer-supplied `boundAt` (06 §1), not the
  chain.
- A lens `category` rule matches classifications from any classifier (06 §6).
  Lenses may need to name the classifiers they trust.

## Publishing history

These specs will be published on chain as documents in the format of
[09](specs/09-documents.md): each file will be the document
`overlay.social/specs/NN`, each revision a signed post, and every later fix
a new revision that points at the one before.

The plan:

- Revision 1 of specs 00–08 will be their earlier text, as of commit
  `5687a08` in this repository, so the chain keeps a record of what that
  version claimed.
- Revision 2 of specs 00–08, and revision 1 of spec 09, will be the text of
  this revision of the repository.
- The publishing identity key and the txid of each published revision will be
  listed here once they exist.

Nothing has been published on chain yet.

## License

Open BSV License v5. See [`LICENSE`](LICENSE). The specs, and anything
derived from them, are usable only on the Bitcoin SV blockchain, by design.
