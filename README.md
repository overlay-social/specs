# overlay.social specs

Open specifications for **sovereign, BRC-100-native social applications on BSV**.

This repository documents the conventions that the [overlay.peck.to](https://overlay.peck.to) reference deployment actually runs, plus the roadmap toward a federated overlay layer. It is deliberately split so you can tell the two apart at a glance.

> **Honesty contract.** Every spec file declares a `Status:` of `IMPLEMENTED`, `PARTIAL`, or `ROADMAP` in its header. If it says `IMPLEMENTED`, there is code running in production against it today. If it says `ROADMAP`, it is a design we intend to build - do not assume an endpoint exists because it is described here.

## What runs today (IMPLEMENTED)

- **Bitcoin Schema admission.** Content is written as `B + MAP + AIP` pushes inside `OP_FALSE OP_RETURN`, signed with canonical AIP (BSM-compact today). Any Bitcom-aware indexer can read it. See [`00-content-bitcoin-schema.md`](specs/00-content-bitcoin-schema.md).
- **Token-state decode (ProfileToken / HandleToken).** Identity entities live as 1Sat UTXOs with a three-layer envelope (ord-inscription / MAP tags / sCrypt state). The overlay decodes `@prop` state out of the locking script and tracks the canonical, unspent token per subject. See [`01-token-state.md`](specs/01-token-state.md).
- **Identity projection - the killer feature.** A single batch call resolves a whole feed page of authors to their canonical ProfileToken state, and the renderer defensively overwrites display name / avatar / handle. SSO-feel across apps with no server session. See [`02-identity-projection.md`](specs/02-identity-projection.md).
- **Topic-rooted overlay state.** Each topic manager (`tm_peck-bio-profile`, `tm_peck-bio-handle`, `tm_peck`) admits outputs and serves REST hydration. See [`03-overlay-topics.md`](specs/03-overlay-topics.md).

## What is designed but not yet built (ROADMAP)

- **Federated overlay sync** - GASP / subtree-rooted topic Merkle trees, N-of-M peer acceptance, idempotent submission, on-chain state-root anchoring. See [`04-federation-roadmap.md`](specs/04-federation-roadmap.md).
- **Additional write channels & token economics** - input-script-data (payment-coupled actions), PostToken marketplace, paywall via BRC-104 channels, Teranode Kafka direct-consume, the Bitcoin-Schema hybrid bridge for token-state posts. See [`05-write-channels-roadmap.md`](specs/05-write-channels-roadmap.md).

## Reference deployment & data reality

The live reference is overlay.peck.to, consumed by peck-web (peck.to). As of this writing only a handful of ProfileToken owners exist on mainnet; most feed authors are bare P2PKH addresses with **no** ProfileToken. The identity projection is built to degrade gracefully for them - unresolved authors are simply omitted and the renderer falls back to a truncated address. Read [`02-identity-projection.md`](specs/02-identity-projection.md) data-reality section before assuming a full identity graph exists.

## Layer map

```
Application        peck.to / peck.bio / peck.ink / Margin
Read/sync          overlay (BRC-22 topic managers, REST hydration)   <- single overlay today; federation = roadmap
Write channels     OP_RETURN Bitcoin Schema (live) / token-state (live) / input-script-data (roadmap)
Identity/content   BRC-100 / BRC-43 / BRC-52 / BRC-77 / BRC-104
Substrate          BSV mainnet
```

## License

MIT.
