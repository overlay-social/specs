# 04 — Federation

- Status: ROADMAP. Nothing in this file runs. One overlay (overlay.peck.to) serves state today.
- Revision: 2
- Date: 2026-10-26

> **In short.** Today one overlay server answers for everyone, which makes it
> both a bottleneck and a party you have to trust. Federation means several
> independent overlays that copy each other's topic state and check each
> other against the commitments they publish on chain. This file records the
> intended design so that today's single-overlay protocol leaves room for it.
> Do not assume any endpoint below exists.

## 1. What already exists to build on

- Per-topic state roots and their on-chain anchors ([08](08-state-anchoring.md)).
  Anyone holding the same admitted set can recompute a root and compare it
  with the anchor.
- App-neutral topic managers ([03](03-overlay-topics.md)): a second overlay
  running the same rules admits the same outputs.

## 2. Planned mechanics

- **Inclusion proofs.** Replace or supplement the flat state root with a
  Merkle tree over the sorted outpoint set, so a peer or client can prove one
  output is in a topic's state without the whole set. Planned endpoints:
  `/v1/topic/:topic/snapshot?block=N`, `/delta`.
- **Outpoint-set listing.** Serve the outpoint set a root is computed from,
  so third parties can recompute it ([08 §4](08-state-anchoring.md)).
- **Peer sync.** GASP-style or subtree-based sync of topic state between
  overlays, so a new overlay bootstraps without rescanning the chain.
- **N-of-M acceptance.** An app may require M overlays to agree before it
  treats a record as authoritative. An overlay that publishes a divergent root
  is detectable and can be dropped.
- **Idempotent submission.** Every submission carries a content hash or
  nonce, and a repeat returns the original answer (dedup window of at least
  24 hours). Today a repeat returns an empty result ([03 §2](03-overlay-topics.md)).
- **Per-peer anchoring.** Each peer anchors its own roots under its own
  `anchor_key` and audits the others.

## 3. Open questions

Subtree format across overlays; anchor cadence (per block or per N changes);
peer discovery (out-of-band list, later an on-chain registry); open or
permissioned membership; resolving disagreement (walk the tree down to the
divergent leaves).

## 4. Formal grounding (informative)

The convergence and equilibrium claims of a federation are meant to be argued
in game-theoretic terms (bounded conformance, resilient equilibria,
partition reconciliation). This is a direction for the design, not a property
of running code, and must not be cited as an implemented guarantee.
