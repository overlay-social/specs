# 07 — Friend: mutual-consent relationships

- Status: IMPLEMENTED
- Revision: 2
- Date: 2026-10-26

> **In short.** Following someone is one-way; being friends takes both
> people. Each person writes a small signed "friend" record pointing at the
> other. When both records exist, the two are friends, and apps can unlock
> things that need consent, such as private messages. Either side can later
> withdraw their own record; neither can speak for the other.

## 0. Why friend is not follow

Follow is one-way interest. Friend is a mutual relationship that gates
person-to-person features. In the older BAP-era protocol a friend record
existed to publish a messaging key. Under BRC-100 the two identity roots
already give a shared key (BRC-42), so the record's job is only consent and
discovery.

## 1. Record (normative)

The shared envelope of [06 §0](06-identity-attestations.md), protocol name
`friend`:

```
preimage keys: app type schema_version identity peer serial
type      friend | unfriend
identity  the sender's identity root (66 hex)
peer      the other person's identity root (66 hex)
sig       the sender's BRC-3 signature, protocol friend, key ID serial
```

`identity` MUST differ from `peer`; both match `^0[23][0-9a-fA-F]{64}$`. A
record carries only the sender's signature: you can declare or withdraw your
own side, never your peer's.

## 2. Mutuality (normative)

```
friend(A→B)                                   request
friend(B→A)                                   acceptance
ACTIVE  ⇔  both directions admitted and not withdrawn
```

There is no separate acceptance record; the second direction is the
acceptance. Each side signs from their own wallet, in their own time.

## 3. Withdrawal (normative)

`type unfriend` withdraws the sender's direction: every earlier `friend` record
from `identity` to `peer` stops counting. The peer's direction is untouched.
If A unfriends B, B's consent stands, A sees an incoming request, and one new
`friend(A→B)` restores the pair. Withdrawing a request that was never
accepted is the same operation.

"Earlier" is admission order on the overlay. An `unfriend` record itself is
never part of the active set.

## 4. Read API

```
GET /v1/friends/:key        # key = identity root (66 hex)
→ { "status": "ok", "address": "<key>",
    "light": { "mutual":      [{ "peer": "<key>", "handle": "<handle>" | null }],
               "pending_in":  [ ... ],
               "pending_out": [ ... ] },
    "outgoing": <n>, "incoming": <n>,
    "data": { "outgoing": [...], "incoming": [...] } }   # legacy BAP-era rows
```

`handle` comes from the canonical light handle claim
([06 §3](06-identity-attestations.md)). Incoming friend records also appear as
`friend_request`, or `friend_accepted` when the recipient had already sent a
request, in `GET /v1/notifications/:address`.

## 5. Legacy Bitcoin Schema compatibility

The record is Bitcoin Schema: the same MAP envelope and the same
`type friend` word, so any MAP indexer parses it. It carries
`identity`/`peer`/`sig` instead of the BAP era's `bapID`/`publicKey`. Legacy
rows are indexed and shown for display only. They never count toward
`mutual`, and there is no mapping between the two eras' identities.

## 6. What friendship gates (informative)

The reference apps allow direct messages only between mutual friends. A
message is encrypted by the sender's wallet to the recipient (BRC-2
encryption under a BRC-42 key, protocol `[2, 'peck dm']`, key ID `'1'`,
counterparty the other identity root) and carried by a message box with
mutual authentication (BRC-103/104). The consent is on chain; the messages
are not. The check is done by the clients today; filtering at the
recipient's message box is planned.
