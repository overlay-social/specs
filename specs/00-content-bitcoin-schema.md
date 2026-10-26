# 00 — Content records (Bitcoin Schema)

- Status: IMPLEMENTED
- Revision: 2
- Date: 2026-10-26

> **In short.** A post, a like, a follow or a chat message is written into a
> Bitcoin transaction as a small, signed data record that any app can read.
> This spec says exactly how that record is laid out, which fields mean what,
> how one post points at another (reply, quote, repost), and how the author's
> signature is made and checked. An app that follows it can read every other
> app's posts, and its own posts show up everywhere.

Key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are used as in RFC 2119
and RFC 8174. Terms are defined in [`GLOSSARY.md`](../GLOSSARY.md).

## 1. Output shape (normative)

A content record is one `OP_FALSE OP_RETURN` output whose data pushes form
sections separated by a pipe push:

```
OP_FALSE OP_RETURN
  <B>    <data> <media-type> <encoding> [<filename>]        # optional
  |
  <MAP>  SET <key1> <val1> <key2> <val2> ...                # required
  [ | <MAP> ADD tags <tag1> <tag2> ... ]                    # optional
  |
  <AIP>  <algorithm> <signer> <signature>                   # see §7
```

| Prefix | Value (ASCII push) |
|---|---|
| B | `19HxigV4QyBv3tHpQVcUEQyq1pzZVdoAut` |
| MAP | `1PuQa7K62MiKCtssSLKy1kh56WWU7MtUR5` |
| AIP | `15PciHG22SNLQJXMoSUaWVi7WSqc7hCfva` |

Push encoding:

- Every push MUST use the minimal push opcode for its length (a length byte
  below 76, else `OP_PUSHDATA1/2/4`).
- The separator MUST be the one-byte data push `01 7c`. A bare `0x7c` byte is
  the opcode `OP_SWAP`, not data.
- No push other than a separator may consist of the single byte `0x7c`.
  Writers SHOULD NOT write a push whose content is the two ASCII characters
  `7c`: the reference overlay's parser also treats that as a separator.
- Writers SHOULD NOT write empty pushes. Readers MUST treat an `OP_0` after
  `OP_RETURN` as an empty push that keeps its position; MAP pairs are
  positional, and skipping it shifts every later pair.
- MAP keys MUST be written bare (`app`, not a prefixed form). The reference
  indexer also reads the legacy prefixed form `2/peck.to-social/v1.<key>`;
  the overlay's admission does not.

One transaction SHOULD carry at most one content record. The reference read
model keys posts by txid, so a second record in the same transaction is not
served as a second post.

## 2. B section

`B <data> <media-type> <encoding> [<filename>]`.

- Text: media type `text/markdown` (or `text/plain` for chat messages),
  encoding `utf-8`. The SDK writes the filename `post.md` for text posts.
- Binary media: the raw bytes, their media type, encoding `binary`, and a
  filename.
- When a record carries both a B body and a MAP `content` value, the
  reference read model shows MAP `content` as the text and falls back to the
  B body.

## 3. MAP keys

`app` and `type` are REQUIRED in every `SET` section.

| Key | Meaning |
|---|---|
| `app` | The app that wrote the record (`peck.to`, `peck.press`, ...). Free text. |
| `type` | The action, see §3.1. |
| `content` | Inline text. The SDK repeats a text post's B body here. |
| `kind` | Content sub-class, see §5. |
| `context`, `tx`, `subcontext`, `reply`, `thread_root_tx` | References to other posts, see §4. |
| `channel` | A public channel slug, written with `context channel`. |
| `mention` | Comma-separated posting addresses (or keys) of mentioned authors. |
| `lat`, `lng`, `alt`, `geohash`, `category`, `title` | Location, see §6. |
| `address`, `handle` | The target of `follow` / `unfollow`. |
| `paymail`, `value`, `linkage` | A tip (`type payment`): recipient, satoshis, BRC-29 key linkage. |
| `display_name`, `avatar`, `bio`, `cert_ref` | The older `type profile` record. New writers use [06](06-identity-attestations.md). |

Readers MUST ignore keys they do not know.

### 3.1 Types

The reference topic manager `tm_social-content` admits an output when it has
a MAP `SET` section whose `type` is one of:

`post`, `reply`, `repost`, `like`, `unlike`, `follow`, `unfollow`, `friend`,
`unfriend`, `message`, `payment`, `profile`, `pin`, `channel`, `block`,
`unblock`, `mute`, `unmute`, `function`, `function_call`,
`function_response`, or any type beginning with `registry:`.

Writers SHOULD write `type post` for posts, replies and pins, and let the
reader classify them (§4, §6). `reply` and `pin` are accepted for older
writers. `tag` (a retroactive label on a post: `context tx tx <txid> tags
<a,b>`) is written by the SDK and read by the reference indexer, but the
reference topic manager does not admit it today, so it reaches the reference
index only through the chain indexer.

## 4. References between posts (normative)

| Action | Wire form |
|---|---|
| Reply | `type post`, `context tx`, `tx <parent txid>`. Writers SHOULD also write `reply <parent txid>` for readers that know only that key. |
| Quote | `type repost`, own B body and/or `content`, `tx <target txid>`. |
| Repost | `type repost`, `tx <target txid>`, no content. |
| Like / unlike | `type like` / `type unlike`, `tx <target txid>`. |
| Channel post | `type post`, `context channel`, `channel <slug>`. |

A txid is 64 lowercase hex characters.

Readers MUST classify a record as a reply when `type` is `post`, `context` is
`tx` and `tx` holds a txid. Readers SHOULD also accept these older forms, in
this order of precedence (the first match names the parent):

1. `type post`, `context tx`, `tx` (the canonical form above);
2. `type reply` with `tx` or `contextValue`;
3. `reply_tx`, `replyTx`, `tx_id_reply`, `in_reply_to` or `inReplyTo`;
4. `type post`, `context post` or `context reply`, with `tx` or `contextValue`;
5. a bare `reply` key.

Each value must be a 64-hex txid to count. Readers SHOULD also read
`type post`, `context tx`, `tx <target>`, `subcontext quote` as a quote.

**Thread root.** A post's thread root is its explicit `thread_root_tx` when
that is a txid; otherwise the parent's thread root (or the parent itself when
the parent has none); a post without a parent is its own root.

## 5. `kind`

`kind` names what the content is. Readers MUST compare it lowercase and
trimmed. The reference read model stores it once per txid and never changes
it. Values in use: `longform` (peck.press articles) and `doc` (a document
revision, [09](09-documents.md)). `pin` appears in the read model only as the
marker of the legacy `type pin`; writers MUST NOT write `kind pin`.

## 6. Location

A location is MAP `lat` and `lng` in the post's own `SET` section, in decimal
degrees (WGS84), as plain decimal strings without an exponent. Optional:
`alt` (metres), `geohash` (advisory; no reader takes the position from it),
`category` (lowercase word) and `title` (one line). A pin is a post with a
location; the type stays `post`.

Readers MUST treat a location as usable only when both values parse as
finite numbers, |lat| ≤ 90, |lng| ≤ 180, and they are not exactly (0, 0).
The SDK writes six decimals by default (about 0.1 m). Writers SHOULD let the
user publish a coarser location: a location on chain is public and permanent.

## 7. AIP signature (normative)

The AIP section names an algorithm, a signer and a signature:

```
| <AIP> BRC77         <signing key, 66 hex>     <base64 DER signature>
| <AIP> BITCOIN_ECDSA <address or 66-hex key>   <base64 65-byte compact signature>
```

**Canonical preimage.** The signed message is the concatenation of the
content bytes of every data push from the first push after `OP_RETURN` up to
and including the signer push. Push opcodes and length prefixes are not part
of it, nor are `OP_FALSE` and `OP_RETURN`, nor the signature push. Each
separator contributes its one byte `0x7c`. Binary pushes contribute their raw
bytes.

**BRC77.** digest = SHA-256(preimage). The signature is ECDSA over the
digest, DER-encoded, written as base64 text. The signer push is the
compressed public key in lowercase hex. Verifiers check the signature against
that key.

**BITCOIN_ECDSA.** digest = SHA-256(SHA-256(`0x18` ‖ "Bitcoin Signed
Message:\n" ‖ varint(len(preimage)) ‖ preimage)). The signature is the
65-byte compact (recoverable) form, written as base64 text. Verifiers recover
the key and compare it with the signer push: its P2PKH address, or the key
itself when the signer push is a 66-hex key.

Rules:

- The signature push MUST be the last push that carries data. A verifier
  MUST NOT report a record as verified when a B or MAP section follows the
  signature; anyone could append one to a copied record.
- Writers SHOULD use BRC77 with the BRC-100 wallet key derived as protocol
  `[1, 'identity']`, key ID `'1'`, counterparty `'self'`, so that one person's
  posts carry one key across clients. This is the key the SDK signs with and
  the key a key-binding ([06](06-identity-attestations.md)) binds.
- Older writers (twetch) sign only the B content push under
  `BITCOIN_ECDSA`. The reference indexer accepts that as a fallback. Such a
  signature does not cover the MAP section, so readers MUST NOT rely on it for
  anything MAP says (type, target, `doc`, `rev`, ...).

### 7.1 Test vector (BRC77)

Private key: SHA-256 of the ASCII string `overlay.social test vector 1` =
`12c3e24308083cf47202a92372812767e9bf0149d75dc4a38415af2ce421542e`.
Signer push: `03d7361ef48e4ae1dbf8588227be1fc1c58eb04656cf1c73782d3074396299f0a1`.

Pushes (the record is the document revision of [09 §7](09-documents.md)):

```
19HxigV4QyBv3tHpQVcUEQyq1pzZVdoAut
"# Hello\n\nSecond revision.\n"
text/markdown  utf-8  hello.md
|
1PuQa7K62MiKCtssSLKy1kh56WWU7MtUR5  SET
app overlay.social  type post  kind doc  doc example.org/hello  rev 2
prev 6ea5ee6d1b2326fc9461d91ac7c0517e931225f01fbd2170845b331928d1072c
title Hello  status PUBLISHED
sha256 816bc0b1efa4e3bbfc90c1c9656ffcb5ae02ad98f3e93f5bd6f5b12408a179d9
|
15PciHG22SNLQJXMoSUaWVi7WSqc7hCfva  BRC77
03d7361ef48e4ae1dbf8588227be1fc1c58eb04656cf1c73782d3074396299f0a1
```

- Preimage: 449 bytes, from `3139 4878…` (the B prefix) through the signer
  push.
- SHA-256(preimage) =
  `605388cc2d52242f6e3134940af503c7e1595ed3e3057dc379b3112f62e9f9a8`.
- One valid signature (RFC 6979 deterministic; any valid ECDSA signature
  verifies), base64:
  `MEQCIDz1HQ2/qm5g4MwDdftqGsx1RF6+/JXne8xVJ1k0WycwAiBwS45NuwiJQPwBV5t1qrTaW95z2hsaulgfNRwGhM8+Lg==`

The full 579-byte locking script, in hex, is in [09 §7](09-documents.md).

## 8. Admission and verification in the reference deployment (informative)

- The reference topic manager admits on the MAP type alone (§3.1). It does
  not verify the AIP signature, and a record without one is admitted.
- The reference indexer verifies AIP signatures as in §7 and stores the
  result. Rows that reach the index through the overlay's `/submit` path are
  marked as verified when an AIP section is present, before any check, and
  the flag is merged with OR. The read model's provenance grade is therefore
  not a proof.
- So: a reader for whom authorship matters (a document chain,
  [09](09-documents.md)) MUST verify the signature itself from the
  transaction bytes.

## 9. When to use this channel

Attribution-only, discoverable content in volume: posts, replies, likes,
follows, tags, channel messages, tips, document revisions. Records that carry
ownership or update rules use token state ([01](01-token-state.md)); signed
claims about an identity use [06](06-identity-attestations.md).
