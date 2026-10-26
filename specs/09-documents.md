# 09 — Documents

- Status: PARTIAL. Revisions can be written, admitted, listed and anchored today. The read model that serves a document's history is not built; until it is, readers resolve history from the transactions themselves (§10). This spec becomes IMPLEMENTED when that read model ships.
- Revision: 1
- Date: 2026-10-26

> **In short.** A document, such as one of these specs, is published as a
> chain of revisions. Each revision is a signed post that holds the full text,
> a fingerprint of that text, and a pointer to the revision before it. Nothing
> is ever edited in place: a fix is a new revision, a retraction is a revision
> that says so, and the old text stays readable. Anyone can check that a
> revision is the author's, that its text is intact, and which revision is
> current, and anyone can quote an exact section of an exact revision.

Key words as in RFC 2119 and RFC 8174. Builds on
[00](00-content-bitcoin-schema.md) (record format and signature),
[06](06-identity-attestations.md) (key-binding) and
[08](08-state-anchoring.md) (anchoring).

## 1. Model

- A **document** is named by an author identity and a document id.
- A **revision** is one transaction carrying one revision record (§2). It is
  identified by its txid and never changes.
- The **series** of a document is every valid revision with the same document
  id and the same author identity (§4.1).
- The **history** is the series ordered by its `prev` links from revision 1
  to the current revision (the **head**).

## 2. Revision record (normative)

One `OP_FALSE OP_RETURN` output in the format of
[00 §1](00-content-bitcoin-schema.md):

```
B    <document bytes> <media type> utf-8 [<filename>]
|
MAP  SET  app <app>  type post  kind doc  doc <document id>  rev <n>
          [prev <txid>]  [title <title>]  status <status>  sha256 <hex>
          [source <url>]  [note <one line>]
|
AIP  <algorithm> <signer> <signature>
```

| Key | Rule |
|---|---|
| `app` | REQUIRED. The publishing app, free text. |
| `type` | REQUIRED, exactly `post`. The record is admitted by `tm_social-content` without change. |
| `kind` | REQUIRED, exactly `doc`. |
| `doc` | REQUIRED. Document id: 1–200 characters from `a-z 0-9 . _ ~ / -`, no leading or trailing `/`, no `//`. By convention `<namespace>/<path>` with a DNS name the publisher uses as namespace, e.g. `overlay.social/specs/08`. Nothing verifies the namespace; the author identity is part of the series key (§4.1), so two authors cannot collide. |
| `rev` | REQUIRED. Positive decimal integer without leading zeros. `1` for the first revision. |
| `prev` | REQUIRED when `rev` > 1, and absent when `rev` is 1: the txid (64 lowercase hex) of the preceding revision. |
| `title` | RECOMMENDED. One line. |
| `status` | REQUIRED. See §6. |
| `sha256` | REQUIRED. Lowercase hex SHA-256 of the B data push (§5). |
| `source` | OPTIONAL. An `https` URL of the same text at a pinned commit. |
| `note` | OPTIONAL. One-line change summary. |

Further rules:

- The B section is REQUIRED and carries the complete document. The media
  type SHOULD be `text/markdown`; the encoding MUST be `utf-8`. A filename
  SHOULD be given.
- The AIP section is REQUIRED (§3).
- A revision record MUST NOT carry `content`, `context`, `subcontext`, `tx`,
  `txid`, `ref_txid`, `contextValue`, `reply`, `reply_tx`, `replyTx`,
  `tx_id_reply`, `in_reply_to`, `inReplyTo`, `thread_root_tx` or `channel`.
  Readers that follow [00 §4](00-content-bitcoin-schema.md) would otherwise
  file it as a reply, quote or channel post, and a MAP `content` would replace
  the B text in the read model.
- A transaction MUST NOT carry more than one record with `kind doc`. Readers
  MUST treat a transaction that does as carrying no valid revision.
- The document SHOULD NOT embed base64 data URIs; the read model moves them
  out of the text.

## 3. Valid revision (normative)

A reader MUST treat a record as a valid revision only when all of these hold:

1. It satisfies §2.
2. Its AIP signature verifies over the canonical preimage
   ([00 §7](00-content-bitcoin-schema.md)), under `BRC77` or `BITCOIN_ECDSA`.
   The signature MUST be the last data push. The older form that signs only
   the B content MUST NOT be accepted: it does not cover `doc`, `rev` or
   `prev`, so a signed text could be re-wrapped as a different revision.
3. The reader checked the signature itself from the transaction bytes, not
   from an overlay's verification flag ([00 §8](00-content-bitcoin-schema.md)).

A revision whose `sha256` does not match its B content is still valid in this
sense (the author signed it as it is), but it is marked (§5).

## 4. History (normative)

### 4.1 Author identity and series

The signing key of a revision is the BRC77 signer push, or for
`BITCOIN_ECDSA` the key recovered from the signature. Its **author identity**
is the identity root it is bound to by an active key-binding
([06 §1](06-identity-attestations.md)), and otherwise the signing key itself
(66 lowercase hex).

The series of (author identity *A*, document id *D*) is every valid revision
whose `doc` is *D* and whose author identity is *A*. Revisions by any other
identity are never part of it, whatever their `doc` and `prev` say.

Readers evaluate key-bindings as they currently stand. If a binding is later
revoked, the revisions signed under it leave the series; readers SHOULD show
them as "signer no longer bound" rather than hide them.

### 4.2 Order

When two revisions compete (§4.3), the earlier one wins. Earlier means:

1. a confirmed revision before an unconfirmed one;
2. among confirmed revisions, the lower block height, then the lower
   position of the transaction in its block (as given by its merkle path);
3. among unconfirmed revisions, the lower txid. Any result that depends on an
   unconfirmed revision is provisional.

This order is a property of the chain, so every reader computes the same
history.

### 4.3 Canonical history

1. **Genesis.** The series' revisions with `rev` 1 are genesis candidates.
   The earliest (§4.2) is the canonical revision 1.
2. **Successor.** The successors of revision *r* are the series' revisions
   with `prev` = txid(*r*) and `rev` = `rev`(*r*) + 1. The earliest successor
   is canonical.
3. Repeat step 2 from the canonical successor. The revision with no
   successor is the head.

A revision whose `prev` names a txid outside the series, or whose `rev` is
not one more than its predecessor's, links nowhere and is an **orphan**.

**Forks are shown, not hidden.** Every valid revision of the series that is
not on the canonical history (a losing genesis candidate, a losing successor,
anything that descends from one, an orphan) MUST stay reachable: a reader
MUST NOT drop it silently, and SHOULD show that the document has revisions
outside its canonical history. Only the author (or a holder of the author's
keys) can create a fork.

## 5. Integrity (normative)

`sha256` is the SHA-256 of the bytes of the B data push exactly as they
appear in the transaction: no Unicode normalization, no line-ending change,
no trimming.

- Readers MUST compute it from the transaction bytes. The read model's `text`
  is a rendering (truncated past a limit, media moved out) and MUST NOT be
  hashed instead.
- On a mismatch, a reader MUST mark the revision as failing its integrity
  check wherever it shows it, and MUST NOT drop it silently. The revision
  keeps its place in the history.

## 6. Status, corrections and withdrawal

`status` is a single uppercase word. Defined values:

| Value | Meaning |
|---|---|
| `DRAFT` | Work in progress. |
| `PUBLISHED` | A finished document. |
| `IMPLEMENTED`, `PARTIAL`, `ROADMAP` | For specifications: running code exists, partly exists, or is planned. |
| `WITHDRAWN` | The author retracts the document. |

Readers MUST show other values as given.

- **Correction.** A mistake is corrected by a new revision. Earlier
  revisions stay readable; readers SHOULD show that a newer revision exists.
- **Withdrawal.** A revision with `status WITHDRAWN`. Its text SHOULD say
  why. The document's current status is the head's status, and a later
  revision MAY republish it. Withdrawal removes nothing.

## 7. Worked example

### 7.1 Revision 2 of spec 08

The pushes of the revision that will publish this revision of
[08](08-state-anchoring.md), as revision 2 of `overlay.social/specs/08`.
Content-dependent values are symbolic: *N* is the byte length of the file,
*H* its SHA-256, *P* the txid of revision 1, *K* the signing key, *C* the
commit.

```
script  OP_FALSE OP_RETURN                         00 6a          not in preimage
──── preimage starts ───────────────────────────────────────────────────────
 1  22 "19HxigV4QyBv3tHpQVcUEQyq1pzZVdoAut"        B prefix
 2  4d <N, 2 bytes LE> <N bytes of 08-state-anchoring.md>   OP_PUSHDATA2 (N > 255)
 3  0d "text/markdown"
 4  05 "utf-8"
 5  15 "08-state-anchoring.md"
 6  01 7c                                          separator
 7  22 "1PuQa7K62MiKCtssSLKy1kh56WWU7MtUR5"        MAP prefix
 8  03 "SET"
 9  03 "app"      0e "overlay.social"
10  04 "type"     04 "post"
11  04 "kind"     03 "doc"
12  03 "doc"      17 "overlay.social/specs/08"
13  03 "rev"      01 "2"                           the byte 0x32, not OP_2
14  04 "prev"     40 "<P>"
15  05 "title"    0f "State anchoring"
16  06 "status"   0b "IMPLEMENTED"
17  06 "sha256"   40 "<H>"
18  06 "source"   <len> "https://github.com/overlay-social/specs/blob/<C>/specs/08-state-anchoring.md"
19  04 "note"     <len> "<one line>"
20  01 7c                                          separator
21  22 "15PciHG22SNLQJXMoSUaWVi7WSqc7hCfva"        AIP prefix
22  05 "BRC77"
23  42 "<K, 66 hex>"
──── preimage ends (signer push included) ──────────────────────────────────
24  4c <len> "<base64 DER signature>"              signature push, not in preimage
```

Rows 9–19 each hold two pushes, key then value. The preimage is the
concatenation of the content bytes of every push from row 1 through row 23,
without push opcodes or length prefixes; the BRC77 signature in row 24 is
over the SHA-256 of that preimage.

### 7.2 Complete test vector

A two-revision document `example.org/hello` with a test key, fully computed.
The pushes, preimage, digest and signature are listed in
[00 §7.1](00-content-bitcoin-schema.md). Values:

- document bytes `# Hello\n\nSecond revision.\n` (26 bytes), SHA-256
  `816bc0b1efa4e3bbfc90c1c9656ffcb5ae02ad98f3e93f5bd6f5b12408a179d9`;
- `prev` is a synthetic value (the SHA-256 of the ASCII text
  `placeholder for the txid of revision 1`), since revision 1 is not on
  chain.

The complete 579-byte locking script:

```
006a2231394878696756345179427633744870515663554551797131707a5a56646f4175741a
232048656c6c6f0a0a5365636f6e64207265766973696f6e2e0a0d746578742f6d61726b646f
776e057574662d380868656c6c6f2e6d64017c223150755161374b36324d694b43747373534c
4b79316b683536575755374d7455523503534554036170700e6f7665726c61792e736f636961
6c047479706504706f7374046b696e6403646f6303646f63116578616d706c652e6f72672f68
656c6c6f03726576013204707265764036656135656536643162323332366663393436316439
3161633763303531376539333132323566303166626432313730383435623333313932386431
30373263057469746c650548656c6c6f06737461747573095055424c49534845440673686132
3536403831366263306231656661346533626266633930633163393635366666636235616530
3261643938663365393366356264366635623132343038613137396439017c22313550636948
473232534e4c514a584d6f535561575669375753716337684366766105425243373742303364
3733363165663438653461653164626638353838323237626531666331633538656230343635
366366316337333738326433303734333936323939663061314c604d45514349447a31485132
2f716d3567344d774464667471477378315246362b2f4a586e653878564a316b305779637741
6942775334354e7577694a5150774256357431717254615739357a32687361756c67664e5277
47684d382b4c673d3d
```

(Line breaks are for display only.)

## 8. Quoting and replying (normative)

A post points at a revision with the reference forms of
[00 §4](00-content-bitcoin-schema.md), naming the revision's txid, plus an
optional MAP `section`:

| Action | Wire form |
|---|---|
| Quote a revision or a section of it | `type repost`, own text, `tx <revision txid>`, `section <slug>` |
| Reply to a revision or a section of it | `type post`, `context tx`, `tx <revision txid>`, `reply <revision txid>`, `section <slug>` |

- A quote or reply SHOULD name a revision txid, not the document id, so it
  stays attached to the exact text it is about.
- `section` names a heading of that revision (below). Because a revision
  never changes, a slug never moves. A reader that cannot find the slug
  SHOULD fall back to the whole revision.

**Slug.** Only ATX headings count: a line that starts with one to six `#`
characters followed by a space, outside fenced code blocks (` ``` ` or
`~~~`). The heading text is the rest of the line with trailing `#`
characters and whitespace removed. Then:

1. lowercase it;
2. remove every character that is not a Unicode letter, a Unicode digit, a
   space, `-` or `_`;
3. replace each space with `-`;
4. the first heading with a given slug keeps it; the *n*th repeat gets `-n`
   appended (`-1`, `-2`, ...).

Headings SHOULD NOT contain links: their URL text would enter the slug.

| Heading | Slug |
|---|---|
| `## 2. Anchor transaction (normative)` | `2-anchor-transaction-normative` |
| `### 7.1 Test vector (BRC77)` | `71-test-vector-brc77` |
| `# 08 — State anchoring` | `08--state-anchoring` |
| ``## 5. `kind` `` | `5-kind` |

## 9. Submission and anchoring

Writers SHOULD submit each revision to the overlay with `POST /submit`,
`x-topics: ["tm_social-content"]` ([03 §2](03-overlay-topics.md)). The
revision output then joins the `tm_social-content` state, and the next anchor
of that topic ([08](08-state-anchoring.md)) commits to a set that contains it.

What this gives, and what it does not:

- The revision's own transaction, once mined, is the primary record: its
  block fixes the text and the time, and no overlay can change it.
- The anchor adds the overlay's commitment that it had admitted the revision
  by the anchor's time. An overlay that later served a history without it
  would contradict its own anchor.
- The state root is a flat hash over the whole topic, and the reference
  overlay does not serve the outpoint set yet, so a reader cannot prove one
  revision's inclusion from the anchor alone today ([08 §4](08-state-anchoring.md)).
- A revision that reaches the reference index only from the chain indexer,
  not through `/submit`, is listed but is not in any anchored set.

## 10. Reference deployment today (informative)

- `tm_social-content` admits revision records as ordinary posts, and the
  read model keeps `kind doc`. `GET /v2/feed?kind=doc` (with `author=` to
  narrow) lists revisions; `GET /v2/post/:txid` serves one with its text.
- The read model does not store `doc`, `rev`, `prev`, `status`, `sha256` or
  `section`, and there is no endpoint for a document's history.
- The overlay does not serve raw transactions for posts. Readers fetch a
  revision's transaction from any chain source and verify it with its merkle
  path, then apply §3–§5 themselves.

## 11. Security considerations

- **Who can extend a document.** Anyone holding a signing key bound to the
  author's identity. A stolen key can publish forks and successors that §4
  cannot tell apart from the author's own. Key-bindings written with a null
  revocation outpoint cannot be revoked ([06 §1](06-identity-attestations.md)).
- **Completeness.** An overlay can leave revisions out of what it serves.
  Readers who need completeness also look on chain for records with the same
  `doc`, and SHOULD compare the head they find with the head the overlay
  serves.
- **Appended data.** A copy of a signed record with extra sections appended
  after the signature fails §3, item 2.
- **Size.** The whole document is in one push. Specs are a few KB; the
  reference overlay accepts submissions up to 10 MB.
