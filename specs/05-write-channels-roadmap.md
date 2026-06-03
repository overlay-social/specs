# 05 - Additional write channels & token economics (ROADMAP)

**Status: ROADMAP - designed, not deployed.** None of the channels or token-economics flows below are running. They are recorded so implementers know the intended shape.

## 1. Input-script-data (payment-coupled actions)

Data carried in the unlocking script of a normal P2PKH spend via `<chunk> OP_DROP` pairs before `<sig> <pubkey>`. The data rides the spend, so payment + record are one event. Natural fit: tip+comment, paid-follow, paywall receipts, endorsements. Same canonical AIP preimage rules as OP_RETURN (OP_DROPs are not part of the preimage). Requires indexers to scan input scripts, which legacy Bitcom parsers do not.

## 2. PostToken family & marketplace

Stateful social content as 1Sat tokens (PostToken, and per-app subtypes CanvasToken/EventToken/ChannelToken). `subject` must own a canonical HandleToken to mint. v1 surface: `update`, `transfer`, `burn`, `reply`; edits = burn+mint-child. Marketplace (`sell`, `bid/settle`, `royaltyBps`) and like-as-payment are unbundled to a later wave pending real usage data.

## 3. Paywall via BRC-104 channels

Per-post paywall: AES-encrypt content (UHRP), set `priceSats` in token state; reader pays `owner` (one-shot or via BRC-104 channel), overlay verifies and releases the decryption key. No token UTXO spend per read. (BRC-104 is UHRP content-addressing; the paywall enforcement is a payment-channel sCrypt contract + receipts - do not conflate.)

## 4. Teranode Kafka direct-consume

Migrate the indexer read-side off third-party relays onto Teranode Kafka topics (`validatortxs`, `txmeta`, `subtrees`, `blocksFinal`, invalidation topics). Lower latency, no third-party dependency, native protobuf. Not a blocker; the current relay path works for OP_RETURN.

## 5. Bitcoin-Schema hybrid bridge for token-state posts

"Strawman A": a token-state-update TX MAY include an extra OP_RETURN Bitcoin Schema MAP post (opt-in per TX via a `post_content` field) so strict-v1 consumers see it as an ordinary post, while token-aware consumers parse both layers and link via outpoint. Canonical output order: [0] state UTXO, [1] token-payload OP_RETURN, [2] BS-MAP (only if `post_content` set), [3] change. Indexer binds `token_ref` only when output[0] matches a known token locking script (anti-spoof). Empty/undefined `post_content` means silent update (no feed entry).
