# 06 - Light identity attestations (key-binding, profile, handle, certs)

**Status: IMPLEMENTED** (overlay.peck.to; consumed live by peck.to). This is
the *light* identity family — plain `OP_FALSE OP_RETURN` MAP records with
BRC-3 self-signatures — that complements the token-state identity of
[`01-token-state.md`](01-token-state.md). Where ProfileToken is sovereign
*stateful* identity (spend-to-update), the light family is sovereign
*attested* identity: cheap, wallet-native, no contract toolchain required.
Resolution prefers light attestations and falls back to token-state.

## 0. Shared envelope and signature scheme (normative)

Every record is a MAP `SET` inside `OP_FALSE OP_RETURN`
(MAP prefix `1PuQa7K62MiKCtssSLKy1kh56WWU7MtUR5`), with fields as
`key value` push pairs. All records are **self-signed under BRC-3**:

```
preimage  = UTF-8 bytes of JSON.stringify(core, sortedKeys)
            where core = the record's PREIMAGE fields (see each type)
sig       = createSignature({ data: preimage,
                              protocolID: [1, '<type>'],
                              keyID: <serial> })
verify    = ProtoWallet('anyone').verifySignature({
                data: preimage, signature: sig,
                protocolID: [1, '<type>'], keyID: <serial>,
                counterparty: <identity pubkey> })
```

- `serial` is a per-record random nonce (hex). It doubles as the BRC-42
  key derivation `keyID`, so each record verifies against a distinct
  derived key of the same identity root.
- `schema_version` is `"1"` for everything below.
- `identity` is always the **identity ROOT** pubkey (66-hex compressed,
  `wallet.getPublicKey({identityKey: true})`), never a derived posting key.
- Admission = signature verifies + fields well-formed. Nothing else. Topic
  managers are app-agnostic: any wallet can emit these.

## 1. Key-binding (`type=keybinding`, topic `tm_key-binding`)

Binds a derived **posting key** (what actually signs feed content) to an
identity root, with consent from *both* keys:

```
PREIMAGE: app type schema_version subject certifier serial purpose revocation [boundAt]
fields:   subject    = posting pubkey (the bound key)
          certifier  = identity root pubkey
          purpose    = e.g. "social-posting"
          revocation = outpoint reserved for future revocation ("00…00.0" = none)
          sig        = certifier's BRC-3 sig   (protocolID [1,'keybinding'], keyID=serial)
          subjectsig = subject's signature over the SAME preimage
                       (protocolID [1,'identity'], keyID '1', counterparty 'self')
```

Mutuality is the point: the identity attests "this key posts for me" AND the
key attests "I belong to this identity". One signature alone admits nothing.

**Effect (normative):** resolution collapses bound keys. Anywhere an author,
actor, or subject appears as a bound posting key **or its P2PKH address**,
identity resolution returns the *root's* canonical identity
(`POST /v1/identities/resolve`, both lanes). Earliest unspent binding per
key wins.

## 2. Light profile (`type=profile`, topic `tm_identity-profile`)

```
PREIMAGE: app type schema_version identity display_name avatar bio serial
```

Latest admitted record per identity is canonical. Preferred over legacy
ProfileToken state at resolution (display name / avatar / bio override).

## 3. Light handle claim (`type=handle`, topic `tm_identity-handle`)

```
PREIMAGE: app type schema_version identity handle serial
handle:   /^[a-z0-9_]{1,30}$/
```

**Canonical owner per handle is FIRST-COME**: the earliest admitted, unspent
claim wins a contested handle (Zooko-triangle decision: claimed, not
unique-by-fiat; deterministic and squatting-resistant). The identity's
self-signature proves the claim is genuinely theirs; first-come decides who
wins. `GET /resolve/:handle` resolves through light claims first, legacy
HandleTokens second.

## 4. Cert publication (`type=cert`, topic `tm_identity-cert`)

The identity self-publishes which BRC-52 certificates it wants public
(`identity`, `certifier`, `cert_hash`, optional `platform`). All unspent
publications are listed under the identity; cert *content* verification
happens at resolution, not admission. Bring-your-own-certifier: trust is
federated per certifier key, not centralized.

## 5. Read API (live)

```
POST /v1/identities/resolve     # batch; collapses bound keys (pubkey + address lanes)
GET  /v1/identities             # discovery: canonical identities, newest first
GET  /resolve/:handle           # light claim first, legacy HandleToken fallback
GET  /identity/:pubkey/signing-keys
```

## 6. Born-on-the-overlay bootstrap (informative)

A new user can be created in ONE transaction with three outputs —
key-binding + light profile + light handle — submitted once with all three
topics. One wallet approval, full identity. This is what the reference
onboarding does.

## 7. Relationship to token-state identity

Both families coexist by design. Token-state (ProfileToken/HandleToken)
carries script-enforced update rules; the light family carries
zero-toolchain self-attestation. Resolution order: light attestations
preferred, token-state fallback, bound-key collapse applied to both.
