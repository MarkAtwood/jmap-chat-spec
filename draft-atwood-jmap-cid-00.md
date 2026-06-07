---
title: JMAP Blob Content Identifiers
abbrev: JMAP Blob CID
docname: draft-atwood-jmap-cid-00
category: std
stream: independent

ipr: trust200902

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
  -
    fullname: Mark Atwood
    email: mark@reviewcommit.com

normative:
  RFC2119:
  RFC5234:
  RFC6234:
  RFC8174:
  RFC8620:

informative:
  JMAP-CHAT:
    title: JMAP for Chat
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-00
    date: 2026
  JMAP-FILENODE:
    title: JMAP File Storage Extension
    author:
      fullname: Bron Gondwana
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-filenode-13
    date: 2026
    target: https://datatracker.ietf.org/doc/draft-ietf-jmap-filenode/
  JMAP-BLOBEXT:
    title: "JSON Meta Application Protocol (JMAP) Blob Management Extension"
    author:
      fullname: Bron Gondwana
      role: editor
    seriesinfo:
      RFC: "9404"
    date: 2023
    target: https://www.rfc-editor.org/rfc/rfc9404

--- abstract

This document defines the `urn:ietf:params:jmap:cid` JMAP capability.
When a server advertises this capability, it extends the blob upload
response defined in {{RFC8620}} with a `sha256` field carrying the
SHA-256 digest of the uploaded content. When {{JMAP-FILENODE}} is also
supported, a `sha256` property is added to FileNode objects. These
additions enable clients to verify blob integrity, detect duplicate
content, and use content-addressed storage patterns without downloading
blobs.

--- middle

# Introduction {#introduction}

JMAP ({{RFC8620}}) assigns blobs an opaque, server-assigned `blobId`.
Clients retrieving blobs have no in-band mechanism to verify integrity or
detect duplicates without downloading the full content. Applications such
as file storage ({{JMAP-FILENODE}}), messaging with attachments
({{JMAP-CHAT}}), and any use of JMAP as a content store all benefit from
a standardized way to obtain the SHA-256 digest of a blob alongside its
identifier.

The JMAP Blob Management Extensions ({{JMAP-BLOBEXT}}) define a
`digest:sha-256` property requestable via `Blob/get`, providing a
base64-encoded digest over an arbitrary byte range of a stored blob. This
document addresses the complementary case: returning the SHA-256 digest of
the full blob content at upload time, when the server has just stored the
content, and persisting that digest in FileNode metadata. The two
mechanisms serve different access patterns and are not in conflict.

The `sha256` field defined in this document is also used by {{JMAP-CHAT}}
for its blob upload response; {{JMAP-CHAT}} defers to this document as
the normative definition of that field.

## Conventions Used in This Document

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP 14
{{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals,
as shown here.

The `sha256` field values defined in this document contain a SHA-256
cryptographic hash as defined in {{RFC6234}}, encoded as a lowercase
hexadecimal string of exactly 64 characters.  In ABNF ({{RFC5234}}):

~~~abnf
sha256-value = 64( %x30-39 / %x61-66 )
               ; 64 lowercase hex digits: 0-9 and a-f
~~~

# Capability {#capability}

A server supporting this extension MUST advertise the
`urn:ietf:params:jmap:cid` capability in the JMAP Session resource
({{RFC8620}} Section 2). The value of this capability is an object with
no defined properties in this version of the specification.

Example Session capability object:

~~~json
{
  "capabilities": {
    "urn:ietf:params:jmap:core": { ... },
    "urn:ietf:params:jmap:cid": {}
  }
}
~~~

# Blob Upload Response Extension {#upload}

When a server advertises `urn:ietf:params:jmap:cid`, the upload
response defined in {{RFC8620}} Section 6.1 is extended with one
additional field:

`sha256` (String):
: The SHA-256 digest of the blob content, encoded as a lowercase
  hexadecimal string of exactly 64 characters.

A server advertising this capability MUST store blob content exactly as
received and MUST compute `sha256` from those bytes. A server that
transforms or re-encodes blob content before storage MUST NOT advertise
this capability. This includes servers that perform antivirus,
content-filtering, transcoding, or metadata-stripping transforms that
alter blob bytes before storage.

When this capability is advertised, the `sha256` field MUST be present
in every successful upload response. If the server cannot compute the
digest (for example, due to an internal error during streaming), the
server MUST fail the upload request entirely rather than returning a
response without `sha256`.

Example upload response (field values are illustrative):

~~~json
{
  "blobId":  "f5be7a7b-13f0-4b7c-bead-3c17de6812a2",
  "type":    "image/png",
  "size":    48291,
  "sha256":  "9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08"
}
~~~

## Client Usage {#upload-client}

Upon receiving an upload response, clients SHOULD compute the SHA-256
digest of the bytes that were sent and compare it against the returned
`sha256`. A mismatch indicates transmission corruption or a non-compliant server;
clients SHOULD treat this as a fatal upload error, discard the returned
`blobId`, and surface the failure to the user or calling application.
Clients MAY retry the upload; a persistent mismatch indicates a server
implementation defect. Clients SHOULD log the discrepancy and SHOULD NOT
silently retry indefinitely.

When downloading a blob, clients that stored the `sha256` at upload time
SHOULD recompute the digest of the received bytes and compare. A mismatch
indicates storage corruption or unauthorized modification; clients SHOULD
treat the downloaded content as untrusted and SHOULD surface the
discrepancy to the user or calling application.

## Relationship to JMAP Blob Management Extensions {#upload-vs-blobext}

{{JMAP-BLOBEXT}} defines `digest:sha-256` as a property requestable via
`Blob/get`, returning a base64-encoded digest over a selected byte range
of a blob. The `sha256` field defined in this document differs in three
ways: it is returned unconditionally at upload time (not on demand), it
always covers the full blob, and it uses lowercase hex encoding rather
than base64. Clients MAY use both mechanisms to cross-validate results, provided they
convert encodings before comparing and request the BLOBEXT digest over
the full blob rather than a partial byte range.

# FileNode Extension {#filenode}

When a server advertises both `urn:ietf:params:jmap:cid` and the
{{JMAP-FILENODE}} capability, FileNode objects MUST include the following
additional property:

`sha256` (String|null):
: The SHA-256 digest of the blob associated with this FileNode, encoded
  as a lowercase hexadecimal string of exactly 64 characters. MUST be
  `null` for directory FileNodes (FileNodes with no associated blob). For
  file FileNodes, the server MUST set this to the SHA-256 digest of the
  stored blob content, encoded as defined in {{upload}}, at the time the
  FileNode was created or last updated. A server MAY return
  `null` for a file FileNode whose digest was not computed at the time of
  creation (for example, FileNodes created before this capability was
  enabled), but MUST NOT return `null` for any file FileNode created or
  updated while this capability is active.

When a `FileNode/set` ({{JMAP-FILENODE}}) create or update changes the
associated blob, the server MUST update `sha256` to reflect the new blob
content before returning the response.

## Client Usage {#filenode-client}

When downloading a file blob identified by a FileNode, clients SHOULD
compute the SHA-256 digest of the received bytes and compare it against
the FileNode's `sha256`. A `sha256` value of null has two possible
causes: the FileNode is a directory (which has no associated blob and
should not be downloaded), or the FileNode is a legacy file entry
created before this capability was active. Clients MUST check the
FileNode type to distinguish these cases. For a legacy file FileNode
with null `sha256`, integrity verification via this mechanism is not
available; clients SHOULD treat such downloads as unverified. A mismatch
on a non-null `sha256` indicates storage corruption or modification;
clients SHOULD treat the downloaded content as untrusted and SHOULD
surface the discrepancy to the user or application.

Clients MAY use the `sha256` property to detect duplicate blobs across
FileNodes or between a FileNode blob and a message attachment blob,
without downloading the full content.

# Security Considerations {#security}

## Digest Algorithm Strength

SHA-256 is currently considered cryptographically strong for integrity
verification. This document does not define algorithm negotiation; if
SHA-256 is deprecated in the future, a new capability or a revision of
this document should be defined.

## Integrity vs. Authenticity

The `sha256` field provides integrity verification, not authentication. A
matching digest at upload time confirms that the server received the bytes
the client sent without corruption. A matching digest at download time
confirms that the server returned the bytes it stored without corruption.
It does not protect against a server that intentionally stores or returns
modified content. Clients SHOULD validate using their own locally computed
digest as described in {{upload-client}} and {{filenode-client}}.

## Blob Probing {#blob-probing}

When a server performs cross-account blob deduplication, the `sha256`
field enables clients to correlate blob content across account boundaries
without downloading blobs. A client with read access to another account's
FileNode objects (for example, via a shared folder) can compare FileNode
`sha256` values against locally known content hashes to detect the
presence of specific files in that account.

Servers that perform cross-account deduplication SHOULD ensure that blob
metadata, including `sha256` values in FileNode objects, is not accessible
across account boundaries without explicit sharing permission. Servers
that do not perform cross-account deduplication are not affected by this
risk.

## Upload Timing Side-Channel

When a server performs blob deduplication across accounts, a client may
infer the presence of specific content in another account by timing
upload responses: if identical content is already stored, the server may
respond more quickly (no write needed), and the returned `sha256`
confirms which content was detected. This timing channel does not require
access to FileNode objects and complements the metadata-based probing
risk described in {{blob-probing}}. Servers that perform cross-account
deduplication SHOULD consider uniform-duration upload responses or other
countermeasures if this channel is a concern.

# IANA Considerations {#iana}

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP
Capabilities" registry established by {{RFC8620}}:

Capability Name:
: `urn:ietf:params:jmap:cid`

Intended Use:
: common

Change Controller:
: IETF

Reference:
: This document.

Security and Privacy Considerations:
: See {{security}} of this document.

--- back

# Relationship to Upstream JMAP Specifications {#upstream}

This document is structured as a standalone capability to enable early
deployment without waiting for revisions to existing specifications. The
normative content defined here maps naturally to upstream changes:

- The upload response extension ({{upload}}) is appropriate as an addition
  to the blob upload response in a future RFC 8620 bis.

- The FileNode extension ({{filenode}}) is appropriate as an additional
  property in the FileNode object defined by {{JMAP-FILENODE}}.

Authors of those specifications are encouraged to consider incorporating
this content, at which point this document would be obsoleted.

# Acknowledgements

This document builds on foundational JMAP work: {{RFC8620}} by the JMAP
working group, and {{JMAP-BLOBEXT}} and {{JMAP-FILENODE}} by Bron
Gondwana.
