# 02 - Identity projection (the killer feature)

**Status: IMPLEMENTED** (overlay.peck.to, consumed live by peck.to). Reference: `peck-web/overlay_client.py::resolve_identities` + `peck-web/routers/pages.py::_enrich_with_profile_tokens`.

Identity projection turns raw on-chain authors (P2PKH addresses or pubkeys) into rich, canonical identities at render time - **one batch call per feed page**, with the renderer defensively overwriting display name / avatar / handle. It gives cross-app SSO-feel with no server session and no central identity registry: identity lives in token-state, the overlay just projects it.

## 1. Batch resolve (normative)

```
POST /v1/identities/resolve
body: { "addresses": ["<base58>", ...], "pubkeys": ["<hex>", ...] }   // either or both, max 200 each
-> 200 { "identities": { "<input-key>": {
        "pubkey": "<hex>",
        "address": "<base58>",
        "handle": "<handle|null>",
        "displayName": "<string|null>",
        "avatarRef": "<uhrp://...|null>",
        "profileOutpoint": "<txid.vout>"
      } } }
```

Rules:
- The result is keyed on the **exact input string**: an address-input yields an address-key, a pubkey-input yields a pubkey-key. Feed authors are P2PKH base58 (the `author` column) so pass them as `addresses`. Pubkey-hex subjects go in `pubkeys`.
- Inputs **without** a canonical, unspent ProfileToken are **omitted** from the dict (ghost authors). This is intended, not an error.
- Call it **once per feed page**, never per row.

## 2. Companion lookups (normative)

- `GET /identity/:pubkey` - full bundle for one identity.
- `GET /resolve/:handle` - handle to pubkey + deep-links into consuming apps.
- `GET /v1/bio/profile?subject=<pk> | owner=<pk> | outpoint=<txid.vout>` - raw ProfileToken state.
- `GET /v1/bio/handle/:handle` - HandleToken by handle.
- `GET /cert/verify/:sha` - verify a BRC-52 cert reference.

## 3. Defensive overwrite (normative)

The renderer treats overlay ProfileToken state as canonical and overwrites the post's own display name / avatar / handle - but **the enrichment must never break the feed**. The reference does exactly this: one batch call, overwrite only the keys present, return the input untouched on any error or empty result.

```python
# reference shape - overlay_client.resolve_identities + pages._enrich_with_profile_tokens
async def enrich(posts):
    addrs = [p["author"] for p in posts if p.get("author")]
    idents = await resolve_identities(addresses=addrs)   # {} on any error -> feed never bricks
    for p in posts:
        ident = idents.get(p["author"])
        if not ident:                # ghost author: no ProfileToken yet -> leave fallback
            continue
        if ident.get("displayName"): p["ui_display_name"] = ident["displayName"]
        if ident.get("handle"):      p["ui_handle"]       = ident["handle"]
        if ident.get("avatarRef"):   p["u_avatar_url"]    = uhrp(ident["avatarRef"])
        p["profile_outpoint"] = ident.get("profileOutpoint")
    return posts
```

## 4. Why this is the killer feature

- **No server session.** The identity is projected from chain state on every render; any app reading the same overlay sees the same identity. That is SSO without a session store.
- **Sovereign.** The user owns the ProfileToken; the overlay cannot forge displayName/avatar - it only projects what the token says.
- **Composable.** `/resolve/:handle` returns deep-links so peck.bio, peck.to, peck.ink resolve the same handle to their own surfaces.

## 5. Data reality (read this)

As of this writing only ~4 ProfileToken owners exist on mainnet. For every other author `resolve` returns nothing, and the renderer falls back to a truncated address (or a legacy users-table row, where one exists). So today the projection **lights up for a handful of identities and is a graceful no-op for the rest**. It is correct and forward-compatible: as users mint ProfileTokens (or existing v0 profiles are backfilled), they light up automatically with zero renderer changes. Do not design UX that assumes a populated identity graph yet.
