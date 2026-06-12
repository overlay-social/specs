# 08 - On-chain state-root anchoring

**Status: IMPLEMENTED** (overlay.peck.to — every topic anchored and
verifiable today; check `GET /state` and compare `anchor.matchesLive`).
This lifts the anchoring section of
[`04-federation-roadmap.md`](04-federation-roadmap.md) from roadmap to
implemented: overlay-served state is now auditable against Bitcoin without
trusting the overlay.

## 1. Topic state-root (normative)

Per topic, the root commits the SET of currently-live outpoints:

```
rows      = unspent rows of the topic's source-of-truth table
outpoints = lowercase("<txid>") + ":" + vout        # one per row
algorithm = dedupe → ASCII-sort → join("\n") → sha256 → hex
empty set → sha256("") = e3b0c442…
```

Deterministic, order-independent, recomputable by anyone holding the same
admitted set. `GET /v1/topic/:topic/root` serves `{topic, count, stateRoot,
source}`; `GET /state` serves all topics plus their latest anchors.

## 2. Anchor transaction (normative)

Each anchor is a **1Sat ordinal** output carrying the root, plus a MAP
beacon output for Bitcom-style indexers:

```
out[0]  1 sat   ord-inscription, content-type application/json:
                { "p": "overlay.social/anchor",
                  "topic": "<topic>", "root": "<hex64>",
                  "count": <n>, "ts": <unix>,
                  "prev": "<txid.vout of previous anchor>" }   # update only
out[1]  0 sat   OP_RETURN MAP SET:
                app peck.overlay-anchor · type anchor ·
                action mint|update · topic · root · count · ts ·
                anchor_key <owner pubkey> · prev (update only)
```

`action=mint` for a topic's genesis anchor, `action=update` thereafter; the
`prev` field forms a **logical chain** of anchors per topic (each fresh
ordinal references its predecessor by outpoint — no prior-coin spend
required, so anchoring never contends with the topic's own UTXOs).

## 3. Cadence (normative)

The scheduler anchors **on change with a floor**: when a topic's live root
diverges from its latest anchored root, a new anchor is published (at most
once per hour per topic). No change → no anchor → no chain spam. Genesis
anchors fire automatically when a new topic appears.

## 4. Verification (normative)

For any topic:

1. `GET /state` → `{stateRoot, anchor: {root, txid, outpoint, matchesLive}}`.
2. `matchesLive == true` ⇒ the overlay claims served state equals anchored
   state. **Don't trust it — check it:**
3. Fetch the anchor tx (`txid`) from any chain source; read `root` out of
   the inscription. SPV-verify the tx against block headers (BRC-62 BEEF /
   merkle path) — no full node needed.
4. Recompute the root from the overlay's served rows (§1) and compare.

A mismatch between served state and the anchored root is **detectable by
anyone**. This is the audit primitive federation (spec 04) builds on: a
peer overlay that serves divergent state can be isolated by comparing
roots, with the chain as referee.

## 5. Trust model (informative)

- The anchor proves *commitment*, not *correctness of admission*: it pins
  what the overlay was serving at time T, so the overlay cannot silently
  rewrite history or fork its answers per client.
- `anchor_key` identifies the anchor publisher. Today that is the reference
  overlay's server wallet; in federation each peer anchors its own roots
  and cross-audits.
- Anchors are funded by the overlay operator (~one small tx per topic
  change-hour) — sustainability comes from anchoring on change, not on a
  timer.
