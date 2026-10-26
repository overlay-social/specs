# 02 — Identity resolution

- Status: IMPLEMENTED
- Revision: 2
- Date: 2026-10-26

> **In short.** On chain, a post is signed by a key or an address, not by a
> name. Identity resolution turns those keys into the name, @handle and
> picture the person published about themselves, for a whole page of posts in
> one request. Every app that asks the same overlay gets the same answer,
> without a login server or a central user database. Authors who never
> published a profile simply come back unresolved, and apps show a shortened
> key instead.

## 1. Sources and precedence (normative)

An identity is an identity root key (66-hex compressed public key). Its
display fields come from, in order of preference:

1. the canonical light profile and light handle claim
   ([06](06-identity-attestations.md));
2. the canonical ProfileToken and HandleToken ([01](01-token-state.md)).

A light value overrides the token value field by field: `displayName`,
`avatarRef` and `bio` from the light profile, `handle` from the light handle
claim. `profileOutpoint` names the record the profile came from.

**Bound keys collapse.** A posting key bound to an identity root by an
unspent key-binding ([06 §1](06-identity-attestations.md)), or that key's
P2PKH address, resolves to the root's identity. When one key has several
unspent bindings, the one with the lowest `boundAt` wins.

## 2. Batch resolve (normative)

```
POST /v1/identities/resolve
body:  { "pubkeys": ["<66 hex>", ...], "addresses": ["<base58 P2PKH>", ...] }
200:   { "status": "ok", "count": <n>,
         "identities": { "<input string>": {
           "pubkey": "<identity root, hex>",
           "address": "<P2PKH of the root>",
           "handle": "<handle>" | null,
           "displayName": "<string>" | null,
           "avatarRef": "<uhrp://… | https://… | …>" | null,
           "profileOutpoint": "<txid.vout>" | null,
           "bio": "<string>"                       // present only when a light profile has one
         } } }
```

- At least one of `pubkeys`, `addresses` MUST be an array. Each list is
  de-duplicated and capped at 200; entries that are not well-formed are
  dropped silently. Pubkeys are compared lowercase; addresses as given.
- The result is keyed by the exact input string.
- An input that resolves to nothing is omitted. This is the normal case for
  most authors, not an error.
- An address resolves when it is the address of a bound posting key, or the
  address of a ProfileToken `subject`. An address whose identity has only
  light records and no binding does not resolve through the address lane;
  resolve it by key.
- Clients SHOULD call it once per page of posts, never once per post.

## 3. Single lookups

- `GET /identity/:pubkey`: the full bundle for one identity, including its
  bound `signingKeys`.
- `GET /identity/:subject/signing-keys`, `GET /identity/:subject/certs`.
- `GET /resolve/:handle`: handle to identity key (light claim first, then
  HandleToken), with deep links into the reference apps.
- `GET /v1/identities`: canonical identities, newest first.
- `/v2` serves the same identity already joined into every post as an
  `AuthorView` ([03 §6](03-overlay-topics.md)).

## 4. Rendering (informative)

A renderer treats the resolved fields as canonical and overwrites the
display name, handle and picture a post carries, but resolution must never
break a feed: on any error or empty answer, render the post with its own
fields or a shortened key.

```python
async def enrich(posts):
    idents = await resolve(addresses=[p["author"] for p in posts if p.get("author")])  # {} on error
    for p in posts:
        ident = idents.get(p["author"])
        if not ident:
            continue                      # unresolved author: keep the fallback
        for src, dst in (("displayName", "name"), ("handle", "handle"), ("avatarRef", "avatar")):
            if ident.get(src):
                p[dst] = ident[src]
    return posts
```

## 5. Trust (informative)

Every field comes from a record the identity signed (light records) or a
token it controls, so an overlay that served an invented or altered value
would be caught by any client that checks. An overlay can still omit an
identity or serve a stale one. A client that needs certainty fetches the
record named by `profileOutpoint` from the chain and verifies it.
