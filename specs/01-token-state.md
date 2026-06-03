# 01 - Token-state: ownership-bearing entities

**Status: IMPLEMENTED for ProfileToken & HandleToken** (live on mainnet, decoded by overlay.peck.to). PostToken family and marketplace methods are ROADMAP - see [`05-write-channels-roadmap.md`](05-write-channels-roadmap.md).

Identity entities live as 1Sat UTXOs whose locking script encodes both data and cryptographically-enforced update rules. The overlay decodes the current state and tracks the canonical, unspent token per subject.

## 1. Three-layer envelope

```
output value: 1 satoshi
output.lockingScript:
  Layer A - ord-inscription:  OP_FALSE OP_IF "ord" OP_1 "application/json" OP_0 <state-json> OP_ENDIF
  Layer B - MAP tags:         OP_FALSE OP_RETURN MAP SET app/type/subject/... [| AIP]
  Layer C - sCrypt state:     prop encodings + state pushes + OP_PUSH_TX continuation enforcement
```

- **Layer A** gives 1Sat-ordinals indexer compatibility.
- **Layer B** gives Bitcoin Schema indexer routing (no sCrypt eval needed to discover the token).
- **Layer C** is canonical truth: mutable economic/state fields, monotonic `version`, transfer/update enforcement via `OP_PUSH_TX hashOutputs`.

Disagreement between layers means a malformed token (rejected at admission).

## 2. State decode (normative)

The overlay extracts state from the data pushes that precede the contract template. For the live identity tokens:

- **HandleToken** - `{handle, owner, version}`; `handle` matches `/^[a-z0-9_]{1,30}$/`, case-insensitive. One canonical handle per owner.
- **ProfileToken** - `{subject, owner, displayName, avatarRef (uhrp://...), bioPageRef, cert_refs[], version}`. `subject` is the immutable author identityKey; `owner` is the current (derived, transferable) spend authority. `subject != owner` is the load-bearing separation.

## 3. Owner-authority derivation (normative)

`owner` is a derived pubkey, never the root identity key:

```
protocolID:   [2, '<family> owner-authority']   // security level 2 = prompt per use
keyID:        sha256(<outpoint>).slice(0,16)     // per-token scoping
counterparty: 'self'
```

## 4. Canonicality

When multiple tokens claim the same subject/handle, the canonical one is the latest in the parent-chain resolved by **lowest block-height + lex-min outpoint**, NULL-height sorts last. Edits are burn-parent + mint-child (`parent_outpoint` names the original); `contentRef` is immutable.

## 5. Data reality

Only a handful of ProfileToken owners exist on mainnet today. The decode path is live and correct, but the population is small - see the data-reality note in [`02-identity-projection.md`](02-identity-projection.md).
