# JMAP CID — Implementer's Guide

For server and client implementers of `draft-atwood-jmap-cid-00`. Covers
integration patterns, encoding details, and operational decisions that the spec
deliberately leaves to implementations.

Read the draft first. This guide does not re-state normative requirements. It
covers what implementers need to know beyond the wire contract.

---

## How to read this guide

JMAP CID is a small spec: one capability, one upload-response extension, one
FileNode extension, and security considerations. This guide is proportionally
short.

Each section below follows the same shape used in the companion guides:

1. **What the spec defines** — with a draft citation.
2. **What you must decide** — the concrete implementation choice.
3. **Considerations** — trade-offs that inform the choice.
4. **Recommended starting point** — a defensible default.

This guide is non-normative. The draft is the source of truth. Where this guide
and the draft disagree, the draft wins.

---

## A note on RFC 2119 keywords

This guide uses RFC 2119 keywords for clarity. Where the keyword reflects a spec
requirement, the normativity belongs to the spec. Where the keyword reflects
implementer best-practice, deviation does not affect protocol interop. Cite the
spec, not the guide, when claiming normative authority.

---

## 1. What JMAP CID provides

JMAP (RFC 8620) assigns blobs an opaque, server-assigned `blobId`. The `blobId`
is server-scoped, possibly ephemeral, and carries no intrinsic relationship to
the blob's content. Two uploads of identical bytes may produce different
`blobId` values; the same `blobId` may be reused for different content after
garbage collection.

JMAP CID adds a `sha256` field — a content-derived, portable, permanent
identifier computed as the SHA-256 digest of the raw blob bytes. It complements
`blobId`; it does not replace it.

| Property | `blobId` | `sha256` |
|---|---|---|
| Assigned by | Server | Content (deterministic) |
| Scope | Single server | Universal |
| Stability | May be ephemeral; server may reassign | Permanent for given content |
| Portability | Not portable across servers | Portable; same bytes produce same hash everywhere |
| Purpose | Server-internal blob reference | Integrity verification, deduplication, cross-server correlation |

The capability identifier is `urn:ietf:params:jmap:cid`. When a server
advertises it, the upload response gains a `sha256` field and (if JMAP FileNode
is also supported) FileNode objects gain a `sha256` property.

---

## 2. The `sha256` field

### 2.1 Encoding

**What the spec defines.** The `sha256` field is a lowercase hexadecimal string
of exactly 64 characters, representing the 256-bit SHA-256 digest of the raw
blob bytes.

```
9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
```

**What you must decide.** Nothing — the encoding is fully specified. But
implementations must be careful to produce and validate exactly 64 lowercase hex
characters. Common mistakes:

- Uppercase hex (`9F86D081...`) — non-conformant.
- Base64 encoding — that is the JMAP-BLOBEXT encoding, not CID.
- Truncated output — some libraries return shorter strings for digests with
  leading zero bytes if you use integer-to-string conversion instead of
  fixed-width hex formatting.

**Recommended starting point.** Use your language's standard hex-encoding
function with explicit lowercase and zero-padding. Validate on read that the
string matches `^[0-9a-f]{64}$`.

### 2.2 Relationship to JMAP-BLOBEXT `digest:sha-256`

**What the spec defines.** RFC 9404 (JMAP Blob Management Extensions) defines
a `digest:sha-256` property requestable via `Blob/get`. Both CID's `sha256`
and BLOBEXT's `digest:sha-256` are SHA-256 digests of blob content. They differ
in three ways:

| Aspect | CID `sha256` | BLOBEXT `digest:sha-256` |
|---|---|---|
| Encoding | Lowercase hex (64 chars) | Base64 (44 chars) |
| When available | Returned unconditionally at upload time | Requested on demand via `Blob/get` |
| Scope | Always full blob | Configurable byte range (full blob or partial) |

**What you must decide.**

- Whether to support both mechanisms (if you implement both CID and BLOBEXT).
- How to cross-validate when a client uses both.

**Considerations.**

- A client that has a CID `sha256` and wants to validate against a BLOBEXT
  `digest:sha-256` must convert encodings. The underlying bytes are identical;
  only the text representation differs.
- BLOBEXT's byte-range capability serves a different use case (partial-blob
  integrity). CID always covers the full blob.
- If your server supports both, ensure both digests are computed from the same
  stored bytes. A mismatch (after encoding conversion) indicates a server bug.

**Cross-validation example.** A client uploads a blob and receives:

```json
{
  "blobId":  "B-a1b2c3d4",
  "type":    "application/octet-stream",
  "size":    0,
  "sha256":  "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
}
```

(Illustrative; the hash shown is SHA-256 of empty input.)

The client then requests the same blob's digest via `Blob/get`:

```json
["Blob/get", {
  "accountId": "u1234",
  "ids": ["B-a1b2c3d4"],
  "properties": ["digest:sha-256"]
}, "0"]
```

Response:

```json
["Blob/get", {
  "accountId": "u1234",
  "list": [{
    "id": "B-a1b2c3d4",
    "digest:sha-256": "47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU="
  }]
}, "0"]
```

Both values represent the same SHA-256 digest (in this example, the digest of
an empty input). To compare:

- Decode the base64 `digest:sha-256` to raw bytes.
- Hex-encode those bytes as lowercase.
- Compare against the CID `sha256` string.

---

## 3. Upload response integration

### 3.1 Server implementation

**What the spec defines.** When `urn:ietf:params:jmap:cid` is advertised, the
upload response (RFC 8620 Section 6.1) is extended with `sha256`. The server
MUST compute the digest from the bytes as received and MUST NOT advertise the
capability if it transforms blob content before storage.

**What you must decide.**

- Where in your upload pipeline to compute the hash (streaming during receipt
  vs. after storage).
- Whether to store the hash alongside the blob or recompute on demand.

**Considerations.**

- *Streaming computation* (hash bytes as they arrive over the wire) is
  preferred: it avoids a second pass over the data after storage and has
  negligible CPU overhead on modern hardware. SHA-256 throughput on commodity
  CPUs exceeds 500 MB/s.
- *Store the hash*: the hash is 32 bytes. Storing it avoids recomputation for
  FileNode queries, deduplication checks, and download-time verification.
  There is no reason not to store it.
- *Content transformation prohibition*: if your server performs antivirus
  scanning, image transcoding, metadata stripping, or any other byte-level
  transformation before storage, the stored bytes differ from the uploaded
  bytes. You MUST NOT advertise `urn:ietf:params:jmap:cid` in this case. If
  you scan but store originals unchanged, you may advertise it.

**Recommended starting point.** Compute SHA-256 in a streaming fashion during
upload receipt. Store the 32-byte digest in the blob metadata table. Return it
in the upload response. Use the stored digest for all subsequent `sha256`
queries (FileNode, deduplication) without recomputation.

### 3.2 Client-side verification

**What the spec defines.** Clients SHOULD compute SHA-256 of the sent bytes
and compare against the returned `sha256`. A mismatch is a fatal upload error.

**What you must decide.**

- Whether to enforce this check or treat it as advisory.
- How to surface a mismatch to the user.

**Recommended starting point.** Enforce the check. On mismatch: discard the
`blobId`, surface an error ("Upload integrity check failed"), and offer retry.
Log the mismatch with both digests for debugging. Do not silently retry
indefinitely — a persistent mismatch indicates a server bug.

---

## 4. Deduplication patterns

CID's `sha256` enables content-addressed deduplication without downloading
blobs. This section covers practical patterns.

### 4.1 Server-side deduplication

**What the spec defines.** The spec does not mandate deduplication. It provides
the hash that makes deduplication possible.

**What you must decide.**

- Whether to deduplicate at the storage layer (store one copy of identical
  blobs, reference-count).
- Scope: per-account only, or cross-account.
- Whether to expose deduplication status to clients.

**Considerations.**

- *Per-account dedup* is straightforward and has no privacy implications. If
  the same user uploads the same file twice, store it once.
- *Cross-account dedup* saves storage but creates the side-channel risks
  described in the spec's Security Considerations (blob probing, upload timing).
  Implement the countermeasures the spec recommends if you go this route.
- *Reference counting*: when deduplicating, track how many references point to
  each blob. Garbage-collect only when the count reaches zero.

**Recommended starting point.** Implement per-account deduplication. On upload,
check whether a blob with the same `sha256` already exists for this account. If
so, return the existing `blobId` (and the same `sha256`) without storing a
second copy. Cross-account deduplication SHOULD be deferred until storage
pressure justifies the added complexity and security considerations.

### 4.2 Client-side pre-upload deduplication

**What you must decide.** Whether clients compute `sha256` before upload to
check for server-side duplicates (an optimization that avoids uploading bytes
the server already has).

**Considerations.**

- This requires an API to query "does a blob with this sha256 exist?" The CID
  spec does not define such an API. You would need a deployment-specific
  endpoint or convention (e.g., a `Blob/query` filter by `sha256`).
- The optimization matters most for large files (video, archives) where upload
  bandwidth is expensive.
- Without the query API, clients can still benefit from post-upload
  deduplication: upload the blob, get `sha256` back, and use it for
  cross-referencing (e.g., "I already have a FileNode with this hash; no need
  to create another").

**Recommended starting point.** Clients SHOULD compute `sha256` before upload
for integrity verification. Pre-upload deduplication queries are an optional
optimization; implement them only if your deployment provides a lookup
mechanism. Post-upload cross-referencing (comparing returned `sha256` values
against known hashes) is available immediately with no additional API.

---

## 5. FileNode integration

### 5.1 FileNode `sha256` property

**What the spec defines.** When both `urn:ietf:params:jmap:cid` and the
JMAP FileNode capability are advertised, FileNode objects include a `sha256`
property. It is `null` for directories and for legacy file FileNodes created
before the capability was enabled. For all other file FileNodes, it MUST be
the SHA-256 digest of the associated blob.

**What you must decide.**

- Whether to backfill `sha256` for existing FileNode records created before
  the capability was enabled.
- How to handle FileNode migrations (e.g., importing files from another
  system).

**Considerations.**

- *Backfill*: reading every stored blob to compute its digest may be expensive
  for large deployments. The spec permits `null` for legacy FileNodes, so
  backfill is optional. But clients benefit from non-null values — integrity
  verification and deduplication work only when `sha256` is populated.
- *Migration*: when importing FileNodes from another JMAP server that also
  supports CID, the source's `sha256` can be used to verify that the imported
  blob matches the original. This is one of the primary portability benefits.

**Recommended starting point.** Populate `sha256` for all new FileNodes
immediately. Schedule a background backfill job for existing FileNodes,
prioritizing recently accessed files. The backfill can run at low priority;
clients that encounter `null` on a file FileNode should treat it as
"unverified" rather than "corrupt."

### 5.2 Cross-server deduplication and migration

FileNode `sha256` enables a client to detect duplicate content across servers
without downloading. A migration workflow:

1. Client reads `FileNode/get` from server A, obtaining `sha256` for each file.
2. Client reads `FileNode/get` from server B, obtaining `sha256` for each file.
3. Files with matching `sha256` values are identical; no transfer needed.
4. Files present on A but not on B: download from A, upload to B, verify
   `sha256` matches.

This pattern also applies to Chat message attachments: a message attachment
blob on one server and a FileNode blob on another with the same `sha256`
contain identical content.

---

## 6. Chat integration

### 6.1 Message attachments with content verification

**What the spec defines.** The JMAP Chat draft defers to `draft-atwood-jmap-cid-00`
as the normative definition of the `sha256` field in blob upload responses.
When a client uploads an attachment for a chat message, the upload response
includes `sha256`.

**What you must decide.**

- Whether to store the `sha256` alongside the message attachment metadata.
- Whether to verify attachment integrity on download.

**Recommended starting point.** Store the `sha256` from the upload response in
the message attachment metadata. When a recipient downloads the attachment,
the recipient's client SHOULD compute SHA-256 of the received bytes and compare
against the stored `sha256`. A mismatch indicates corruption or tampering in
transit or at rest.

For federated messages: the `sha256` travels with the message metadata. The
receiving server can verify that the blob it fetched from the sending server
matches the declared hash. This provides end-to-end integrity verification
across the federation boundary without requiring the sending server to be
trusted for content integrity.

---

## 7. Security considerations

### 7.1 Integrity, not access control

**What the spec defines.** The `sha256` field provides integrity verification,
not authentication or authorization. A matching digest confirms the bytes are
what was stored; it does not prove who stored them or grant access to the blob.

**What you must decide.** Nothing — this is a design constraint, not a
deployment choice. But implementations must internalize it:

- Knowing a blob's `sha256` MUST NOT grant access to download that blob. Access
  control remains governed by `blobId` and the server's existing authorization
  model.
- Do not implement "download by hash" APIs that bypass blob-level access
  control. If a client presents a `sha256` value, the server must still verify
  the client has access to a `blobId` associated with that hash.
- In cross-account deduplication scenarios, two users may have blobs with the
  same `sha256`. User A knowing the hash MUST NOT allow User A to access
  User B's blob.

### 7.2 Blob probing and upload timing

**What the spec defines.** The spec's Security Considerations describe two
side-channel risks when servers perform cross-account deduplication:

- *Blob probing*: a client with read access to another account's FileNode
  `sha256` values can detect the presence of specific files without downloading
  them.
- *Upload timing*: if the server responds faster when identical content already
  exists (no write needed), a client can infer the presence of specific content
  in another account.

**Recommended starting point.**

- If you implement cross-account deduplication, ensure FileNode `sha256` values
  are not accessible across account boundaries without explicit sharing
  permission.
- Consider uniform-duration upload responses (always write, or always delay
  to a fixed minimum) if timing side-channels are a concern for your threat
  model.
- If cross-account deduplication is not implemented, these risks do not apply.

### 7.3 Algorithm longevity

SHA-256 is currently considered cryptographically strong. The spec does not
define algorithm negotiation; if SHA-256 is deprecated in the future, a new
capability or revision will be needed. Implementations SHOULD NOT attempt to
future-proof by supporting alternative algorithms outside the spec — doing so
creates interoperability fragmentation.

---

## Cross-references to existing guides

| If you are implementing... | Read also... |
|---|---|
| Core JMAP Chat (governance, authorization, identity) | `jmap-chat-implementer-guide.md` |
| Space file storage (backend, scanning, quotas) | `jmap-chat-filenode-guide.md` |
| Federation wire protocol | `jmap-chat-federation-guide.md` |
| Blob Management Extensions (RFC 9404) | `draft-atwood-jmap-cid-00.md` Section 3.2 (Relationship to JMAP Blob Management Extensions) |

This guide focuses on the CID capability and its integration points.
Transport, push, federation, and FileNode operational details live in their
dedicated guides.
