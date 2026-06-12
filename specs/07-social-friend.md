# 07 - Friend: mutual-consent relationships (`tm_social-friend`)

**Status: IMPLEMENTED** (overlay.peck.to; consumed live by peck.to —
friendship gates direct messaging). Friend is a RELATION, not an identity
primitive — hence the `tm_social-*` family — but it uses the same BRC-3
self-attestation envelope as [`06-identity-attestations.md`](06-identity-attestations.md).

## 0. Why friend ≠ follow

Follow is one-way interest. Friend is a **mutual relationship** that gates
person-to-person capabilities (DMs first). In the BAP era the friend record
existed to *publish a messaging public key*; under BRC-100/BRC-42 key
exchange is **implicit** (ECDH between identity roots), so the record's job
reduces to what actually remains: **consent + discovery**.

## 1. Record (normative)

MAP `SET` in `OP_FALSE OP_RETURN`, BRC-3 self-signed (envelope and
verification exactly as in spec 06 §0):

```
PREIMAGE: app type schema_version identity peer serial
fields:   type     = "friend" | "unfriend"
          identity = SENDER identity root pubkey (66-hex compressed)
          peer     = TARGET identity root pubkey
          serial   = random nonce (= BRC-42 keyID for sig)
          sig      = BRC-3 self-signature, protocolID [1,'friend'],
                     counterparty = identity
constraints: identity != peer; both match /^0[23][0-9a-fA-F]{64}$/
```

Only the *sender's* signature exists on a record: you can declare or revoke
**your own** side of a relationship, never your peer's.

## 2. Mutuality (normative)

```
friend(A→B)               = request
friend(B→A)               = accept
ACTIVE  ⇔  both directions admitted and unrevoked
```

Two independent one-way attestations form a pair. There is no distinguished
"accept" record type — the second direction *is* the acceptance. Asynchronous
by construction: each side signs from their own wallet, in their own time.

## 3. Revocation (normative)

`type=unfriend` (same preimage shape, same signature scheme) revokes **the
emitting side's** direction: the overlay marks every unspent
`friend(identity→peer)` row spent. The peer's direction is untouched — if A
unfriends, B's standing consent remains, so A sees an *incoming request*
and can re-accept later with a single new `friend(A→B)`. Withdrawing a
not-yet-accepted request is the same operation.

## 4. Read API (live)

```
GET /v1/friends/:subject        # subject = identity root pubkey
-> { light: { mutual:      [{peer, handle}],
              pending_in:  [{peer, handle}],
              pending_out: [{peer, handle}] },
     data:  { outgoing: [...], incoming: [...] } }   # legacy BAP-era rows
```

`handle` is hydrated from the canonical light handle claim (spec 06 §3).
Incoming friend records also surface as `type=friend_request` items in
`GET /v1/notifications/:address` (resolved through the key-binding layer).

## 5. Legacy Bitcoin Schema compatibility

The record IS Bitcoin Schema — same MAP envelope, same `type=friend`
vocabulary — so any bmap-style indexer parses it; it simply carries
`identity`/`peer`/`sig` instead of the BAP era's `bapID`/`publicKey`
(which existed to publish a key that BRC-42 now derives). Legacy rows are
indexed and served **display-only**: they never count toward mutual, and
there is no cross-era identity mapping.

## 6. What friendship gates (informative)

The reference deployment gates direct messaging on mutual friendship:
DM content is BRC-42-encrypted between the identity roots
(`protocolID [2,'peck dm']`) and transported via a BRC-103 mutual-auth
message-box — the consent handshake lives on-chain, the message content
never does (an optional publish-permanent path, which would also be
Bitcoin-Schema `type=message` compatible, is roadmap). Enforcement today is
client-side; receiver-side filtering against the friend graph is the
intended hardening.
