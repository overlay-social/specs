# 06 — Self-signed records: identity, key-binding, certificates, lenses, classifications

- Status: IMPLEMENTED for §1–§5. §6 (classifications) is PARTIAL: admitted and used by feeds, but not in the anchored state.
- Revision: 2
- Date: 2026-10-26

> **In short.** A person proves who they are on the overlay with small signed
> statements, each written once to the chain: "this is my name and picture",
> "I claim the handle @anna", "this posting key speaks for me", "I want these
> certificates shown". Anyone can check each statement with the person's
> public key, no special software needed. The same signed-statement format
> also carries moderation: published filter rules ("lenses") and labels on
> posts ("classifications"), which readers may choose to apply. Nothing is
> deleted; readers filter.

## 0. Shared envelope and signature (normative)

Every record in this file is one `OP_FALSE OP_RETURN` output holding a single
MAP `SET` section (no B, no AIP). Values are UTF-8 strings.

```
OP_FALSE OP_RETURN  1PuQa7K62MiKCtssSLKy1kh56WWU7MtUR5  SET
  app <app>  type <type>  schema_version 1  <the type's fields>  serial <hex>  sig <hex>
```

- **Preimage.** Take the record's preimage keys (listed per type below) that
  are present, build a JSON object of them, and serialize it with keys sorted
  ascending and no whitespace (`JSON.stringify(core, Object.keys(core).sort())`).
  The preimage is its UTF-8 bytes. `sig` (and `subjectsig`) are never in it.
- **Signature (BRC-3).** The signer's BRC-100 wallet signs with
  `createSignature({ data: preimage, protocolID: [1, <protocol>], keyID: serial })`
  (counterparty left at its default, `anyone`). `sig` is the DER signature in
  hex. Verifiers check it with an `anyone` wallet:
  `verifySignature({ data, signature, protocolID: [1, <protocol>], keyID: serial, counterparty: <signer key> })`.
- **`serial`.** A fresh random hex value per record. It is also the key ID of
  the signature, so every record is signed by a different key derived from
  the same root.
- **`schema_version`** MUST be `1`.
- **Signer key.** The identity root (66-hex compressed key, from
  `getPublicKey({ identityKey: true })`), never a derived posting key, except
  where a type says otherwise.
- **Admission** = the signature verifies and the fields are well-formed.
  Nothing else; any app may write these records.

| Type | Topic | Protocol name | Signer field |
|---|---|---|---|
| `keybinding` | `tm_key-binding` | `keybinding` | `certifier` |
| `profile` | `tm_identity-profile` | `profile` | `identity` |
| `handle` | `tm_identity-handle` | `handle` | `identity` |
| `cert` | `tm_identity-cert` | `certpub` | `identity` |
| `friend`, `unfriend` | `tm_social-friend` | `friend` | `identity` ([07](07-social-friend.md)) |
| `lens` | `tm_social-lens` | `lenses` | `issuer` |
| `classification` | `tm_social-classification` | `classification` | `classifier` |

The protocol names `certpub` and `lenses` differ from their types because a
BRC-43 protocol name must be at least five characters. They are part of every
signature already published and MUST NOT change.

**One transaction, whole identity (informative).** A new user can be created
with one transaction holding three outputs (key-binding, light profile,
light handle), submitted once with all three topics. One wallet approval.

## 1. Key-binding (`type keybinding`)

Binds a posting key (the key that signs content, [00 §7](00-content-bitcoin-schema.md))
to an identity root, with consent from both.

```
preimage keys: app type schema_version subject certifier serial purpose revocation boundAt
subject     posting key (66 hex)
certifier   identity root (66 hex); MUST differ from subject
purpose     e.g. social-posting
revocation  <txid.vout>; the SDK writes 000…000.0 (no revocation possible)
boundAt     optional non-negative integer, set by the writer
sig         certifier's BRC-3 signature, protocol keybinding, key ID serial
subjectsig  the posting key's signature over the same preimage, DER hex,
            verified directly against `subject`
```

The SDK makes `subjectsig` with `createSignature({ protocolID: [1,'identity'],
keyID: '1', counterparty: 'self' })`, the same derivation as its AIP signing
key, so `subject` is the key that signs that person's posts.

Effect (normative):

- Resolution collapses bound keys ([02 §1](02-identity-projection.md)):
  wherever a bound key or its P2PKH address appears, the root's identity is
  returned.
- A later binding for the same (`subject`, `certifier`) pair replaces the
  earlier one.
- When one key has several unspent bindings, the one with the lowest
  `boundAt` wins (records without `boundAt` sort last). `boundAt` is set by
  the writer, not by the chain.
- Spending the `revocation` outpoint unbinds, but only if the overlay tracks
  that output. With the SDK's null outpoint a binding is permanent.

## 2. Light profile (`type profile`)

```
preimage keys: app type schema_version identity display_name avatar bio serial
```

The latest admitted record per `identity` is canonical. It is preferred over
ProfileToken state at resolution.

## 3. Light handle claim (`type handle`)

```
preimage keys: app type schema_version identity handle serial
handle: ^[a-z0-9_]{1,30}$
```

Per handle, the earliest admitted unspent claim is canonical: first come,
first served. The signature proves the claim is the identity's own; the order
of admission decides contested handles. `GET /resolve/:handle` resolves light
claims first and HandleTokens second.

The order is admission order on the reference overlay, not a property of the
chain. Two overlays could disagree on a contested handle; see the README's
open questions.

## 4. Certificate publication (`type cert`)

```
preimage keys: app type schema_version identity certifier cert_hash platform serial
certifier  key of the BRC-52 certificate's issuer
cert_hash  SHA-256 of the certificate, 64 hex
platform   optional label, e.g. github
```

The identity chooses which certificates to make public. All unspent
publications are listed for the identity. The certificate itself is checked
when it is resolved (`GET /identity/:subject/certs`, `GET /cert/verify/:sha`),
not at admission. Readers decide which certifiers they trust.

## 5. Lens (`type lens`)

A lens is a published, signed set of filter rules that a reader may choose to
apply to their own feed. It hides nothing for anyone who does not apply it.

```
preimage keys: app type schema_version issuer scope rules jurisdiction serial
scope         personal | curated | jurisdictional
rules         JSON array of { "match": ..., "value": ..., "action": ..., "label"?: ... }
jurisdiction  optional
match         author | app | txid | tag | geo | category
action        hide | downrank | label
```

- A rule with `match author` MUST carry a P2PKH address; otherwise the whole
  record is invalid.
- A lens is named by its `lens_id`, the txid of its record. A new version is
  a new record with a new `lens_id`.
- Readers apply a lens with `?lens=<lens_id>[,…]` on `/v1/feed` and
  `/v2/feed`. The reference deployment applies `hide` rules on `author`,
  `app`, `txid` and `category`. `downrank`, `label` and the `tag` and `geo`
  matches are stored but not applied.
- `GET /v1/lenses`, `GET /v2/lenses`.

## 6. Classification (`type classification`) — PARTIAL

A signed statement by a classifier (a person or an automated agent) about
what a post is.

```
preimage keys: app type schema_version classifier target categories model serial
target      txid, or txid.vout, of the classified output
categories  JSON array of { "label": <string>, "confidence": <0..1> }
model       optional: what produced the labels
```

Labels are an open set. A classification is data, not policy: a lens rule
with `match category` decides what a reader hides. In the reference
deployment such a rule matches classifications from any classifier.
`tm_social-classification` is not in the anchored state
([08](08-state-anchoring.md)).

## 7. Relation to token-state identity

Both families run side by side. Token state ([01](01-token-state.md)) has
script-enforced update rules; the records here need only a wallet. At
resolution the records here win, token state is the fallback, and bound keys
collapse for both.
