# 04 - Federation (ROADMAP)

**Status: ROADMAP - not implemented.** Today exactly one overlay (overlay.peck.to) serves state. Nothing in this file is running. It documents the intended direction so the single-overlay design leaves room to grow without protocol breakage.

## 1. Why federation

As BSV scales (Teranode era), single-overlay re-derivation from chain becomes uneconomic, and a single overlay is a trust bottleneck. The intended model is a federation of specialized overlays that sync topic-state peer-to-peer and keep each other honest via game theory.

## 2. Planned mechanics

- **Topic-rooted Merkle trees** served at `/v1/topic/{topic}/root`, `/snapshot?block=N`, `/delta`.
- **GASP / subtree-based sync** between overlays, mirroring Teranode's subtree architecture; near-constant-time bootstrap regardless of chain depth.
- **On-chain state-root anchoring** - overlays periodically commit topic roots to chain (OP_RETURN or token-state) so overlay-derived state is itself attested.
- **N-of-M peer acceptance** - apps may require agreement among M peers before treating a record as authoritative; an overlay publishing an inconsistent root becomes detectable and isolatable.
- **Idempotent submission** - every submission carries a content-hash/nonce; repeat submissions return the same response (>=24h dedup window), eliminating the missed-response failure mode.

## 3. Formal grounding (reference, not a dependency)

The federation's convergence/equilibrium claims are intended to be reasoned about under an automata-theoretic / game-theoretic frame (bounded conformance, resilient Nash equilibria, Merkle-proof state representations, partition reconciliation). This is a theoretical anchor for the design - it is **not** running code and must not be cited as an implemented guarantee.

## 4. Open questions

Cross-overlay subtree-format standardization; anchor cadence (per-block vs per-N); peer discovery (out-of-band registry -> token-state registry contract); federation membership (open vs permissioned); disagreement resolution (Merkle-walk to divergent leaves).
