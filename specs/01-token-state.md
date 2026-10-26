# 01 — Token-state identity (ProfileToken, HandleToken)

- Status: IMPLEMENTED for ProfileToken and HandleToken decoding and canonical selection. Cross-layer consistency checks are ROADMAP. The PostToken family is ROADMAP ([05](05-write-channels-roadmap.md)).
- Revision: 2
- Date: 2026-10-26

> **In short.** Some things a person owns, such as their profile or their
> @handle, can live as a one-satoshi coin whose script holds the current
> values and the rules for changing them. Updating the profile means spending
> the coin and creating its successor. This spec says how a reader decodes
> such a coin and, when several coins claim the same person or handle, which
> one counts. Most new identities use the lighter signed records of
> [06](06-identity-attestations.md); this family is the older, heavier one.

## 1. Envelope

A token is a 1-satoshi output. The specified envelope has three layers:

```
Layer A  ord inscription:  OP_FALSE OP_IF "ord" OP_1 "application/json" OP_0 <state JSON> OP_ENDIF
Layer B  MAP beacon:       OP_FALSE OP_RETURN MAP SET app <app> type profile|handle subject <key> ...
Layer C  sCrypt state:     contract @prop state pushes + the contract's update rules
```

- Layer A makes the token visible to 1Sat-ordinals indexers.
- Layer B lets Bitcoin Schema indexers find the token without evaluating the
  contract.
- Layer C is the contract: the authoritative state and its update rules.

**What is on chain today.** The live minter (peck.bio) writes Layer C in the
token output and Layer B as a separate `OP_RETURN` output in the same
transaction. Layer A and an inline Layer B are specified but not emitted.

## 2. Admission and decoding (normative)

- `tm_social-profile` admits the 1-satoshi token output of a transaction that
  carries a MAP beacon with `type profile` and a `subject`. `tm_social-handle`
  does the same for a beacon with `type handle` and a `handle`. The beacon is
  not itself admitted.
- Readers decode Layer C first. When the contract template does not match,
  they fall back to Layer A, then to the beacon's `subject` and `version`.
- The reference overlay does not reject a token whose layers disagree. A
  consistency check is ROADMAP.

Decoded state:

- **ProfileToken**: `subject`, `owner`, `version`, `displayName`, `bio`,
  `avatarRef`, `certRefs[]`, `handle`, `nickname`. `subject` is the immutable
  identity key the profile describes. `owner` is the key that may spend
  (update or transfer) the token; it defaults to `subject` when absent.
- **HandleToken**: `handle`, `owner`, `version`. `handle` matches
  `^[a-z0-9_]{1,30}$`.

## 3. Owner key (recommendation)

`owner` SHOULD be a key derived for the token, not the identity root, so that
spending authority can move without moving identity. The overlay does not
check how `owner` was derived.

## 4. Canonical token (normative)

Among the unspent tokens:

- **ProfileToken, per `subject`**: the highest `version` wins. Ties go to the
  lowest mint block height (unconfirmed sorts last), then the lowest txid.
  The version rule keeps the latest update canonical even when a reader
  missed the spend of the previous one.
- **HandleToken, per `handle`**: the lowest mint block height wins
  (unconfirmed sorts last), then the lowest txid.

An edit spends the previous token and creates its successor.

## 5. Read API

`GET /v1/bio/profile?subject=|owner=|outpoint=`, `GET /v1/bio/handle/:handle`,
`GET /v1/bio/handle?owner=`. Resolution across all identity families is
[02](02-identity-projection.md).

## 6. Data reality (informative)

On 2026-10-26 the reference overlay's `/state` counted 5 unspent ProfileTokens
and 20 unspent HandleTokens on mainnet, against 14 light profiles
([06](06-identity-attestations.md)).
