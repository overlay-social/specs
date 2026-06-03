# 00 - Content: Bitcoin Schema admission

**Status: IMPLEMENTED** (overlay.peck.to, peck-indexer-go AIP verifier, topic `tm_peck`).

The baseline write channel. Content is carried in an `OP_FALSE OP_RETURN` output using the Bitcoin Schema `B + MAP + AIP` push stack. Every Bitcom-aware indexer - and overlay.peck.to - can read it.

## 1. Output shape

```
OP_FALSE OP_RETURN
  <B_PREFIX>  <content>  <media-type>  <encoding>
  | (0x01 0x7c)
  <MAP_PREFIX>  SET  <key1> <val1>  <key2> <val2> ...
  | (0x01 0x7c)
  <AIP_PREFIX>  BITCOIN_ECDSA  <signing-address>  <signature>
```

## 2. MAP key namespace

Keys are emitted both bare and BRC-43-namespaced (`2/peck.to-social/v1.<key>`). Core keys:

| Key | Meaning |
|---|---|
| `app` | App discriminator (`peck.to`, `peck.ink`, ...) |
| `type` | `post`, `reply`, `repost`, `like`, `follow`, `tag`, `message` |
| `content` | Inline UTF-8 content (or via `B` payload) |
| `reply_tx` | Parent txid (threading) |
| `context` | URN cross-ref (`tx:`, `url:`, `image:`) |
| `cert_ref` | BRC-52 cert txid |

## 3. Canonical AIP preimage (normative)

The AIP signature covers **every preceding pushdata-content byte** (B + MAP + AIP-header up to and including `signing-address`), with pipes counted as `0x7c` data bytes. Opcodes (`OP_FALSE`, `OP_RETURN`, `OP_DROP`) are NOT part of the preimage - the verifier walks the data pushes only. This rule is channel-uniform: it is identical for OP_RETURN and for the roadmap input-script channel.

Today the signature algorithm is BSM-compact (`BITCOIN_ECDSA`). A BRC-77 algorithm marker is reserved for v2.

## 4. Pushdata gotchas (normative)

- Push every chunk via the SDK's `Script.writeBin` wrapper. Do **not** hand-prepend length bytes - `@bsv/sdk` silently mis-encodes raw pushdata.
- The pipe separator MUST be pushed as `01 7c <pushdata>`, never a raw `0x7c` byte (that decodes as `OP_SWAP`).

## 5. When to use this channel

High-volume, attribution-only, discoverable-by-everyone content: posts, likes, follows, tags, channel messages. For ownership-bearing entities, use token-state ([`01-token-state.md`](01-token-state.md)).
