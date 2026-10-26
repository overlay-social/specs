# Glossary

Plain explanations of the terms the specs use. Where a term names a BSV
standard, the number refers to the BRC series at [bsv.brc.dev](https://bsv.brc.dev).

**Anchor.** A small Bitcoin transaction in which an overlay writes the state
root of one topic, so its claims can later be checked against the chain
([08](specs/08-state-anchoring.md)).

**AIP (Author Identity Protocol).** The signature section at the end of a
content record. It names the signing algorithm, the signer and the signature
([00 §7](specs/00-content-bitcoin-schema.md)).

**Atomic BEEF / BEEF (BRC-95 / BRC-62).** A file format for handing someone a
transaction together with the proofs that its inputs are in blocks, so they
can check it without running a full node.

**B.** The section of a content record that carries the body: text or a
file, its media type and encoding.

**Bitcoin Schema.** A shared convention for writing social data (B, MAP and
AIP sections in one data output) that many BSV apps read.

**BRC.** "Bitcoin Request for Comment", the numbered BSV standards. The ones
used here: BRC-3 (signatures anyone can verify), BRC-22 (overlay topic
managers and submission), BRC-26 (UHRP content addresses), BRC-29 (payments
to derived keys), BRC-42 (deriving many keys from one), BRC-43 (protocol
names, key IDs and security levels for derived keys), BRC-52 (identity
certificates), BRC-62/95 (BEEF), BRC-77 (message signatures), BRC-100 (the
interface between an app and a user's wallet), BRC-103/104 (mutual
authentication, over HTTP).

**BRC-100 wallet.** The user's wallet software. Apps ask it to sign, encrypt
and pay; the app never sees a private key.

**Canonical.** The one record that counts when several compete, chosen by a
fixed rule that every reader applies the same way.

**Content record.** A post, reply, like, follow, message or similar, written
as one data output ([00](specs/00-content-bitcoin-schema.md)).

**Document, revision, series, head.** A document is published as a chain of
revisions. The series is all of one author's revisions of one document; the
head is the current one ([09](specs/09-documents.md)).

**Fork.** Two revisions that both claim the same predecessor. The earlier one
on chain wins; the other stays visible.

**Identity root.** The main public key of a person's wallet. Other keys
(posting keys, per-record keys) are derived from it.

**Key-binding.** A record, signed by both keys, that says a posting key
speaks for an identity root ([06 §1](specs/06-identity-attestations.md)).

**Kind.** An optional label for what a post is, such as `longform` or `doc`.

**Lens.** A published, signed set of filter rules a reader may apply to their
own feed. It hides nothing for anyone else.

**Lookup service.** The part of an overlay that indexes a topic's admitted
outputs and answers questions about them.

**MAP.** The section of a content record that holds key/value fields such as
`app`, `type` and `tx`.

**Merkle path (SPV).** The short proof that a transaction is in a block,
checked against block headers only.

**1Sat ordinal.** A one-satoshi output that carries data (an "inscription")
and can be tracked and transferred as a token.

**OP_RETURN output.** A transaction output that carries data and can never be
spent.

**Outpoint.** One output of one transaction, written `txid.vout` or
`txid:vout`.

**Overlay.** A server that accepts transactions, keeps the outputs that fit
its topics' rules, and serves them over HTTP ([03](specs/03-overlay-topics.md)).

**P2PKH address.** The familiar Bitcoin address form (`1…`), derived from a
public key.

**Posting key.** The key that signs a person's content records. It is derived
from the identity root and bound to it.

**Preimage.** The exact bytes a signature covers.

**Read model.** The overlay's ready-to-render answer format (`/v2`,
contract `peck-view/v1`), with authors and counts already joined in.

**Slug.** A heading turned into a short, fixed identifier, used to quote a
section ([09 §8](specs/09-documents.md)).

**State root.** One SHA-256 fingerprint of all of a topic's current
outputs ([08 §1](specs/08-state-anchoring.md)).

**STEAK.** The overlay's answer to a submission: for each topic, which
outputs it admitted.

**Topic, topic manager.** A topic is a named set of outputs that follow one
set of rules; the topic manager applies the rules.

**Txid.** A transaction's identifier, 64 hex characters.

**UHRP.** A way to name a file by the hash of its contents (BRC-26).

**Unspent.** An output that no later transaction has used as an input. For
records, "unspent" means still current.
