---
title: JMAP Graph Provenance Records
abbrev: JMAP Graph Provenance
docname: draft-atwood-jmap-graph-provenance-00
category: std
stream: IETF

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
  RFC8174:
  RFC3987:
  RFC4648:
  RFC8620:
  RFC9670:
  JMAP-GRAPH:
    title: "JMAP for Graphs"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-graph-00
    date: 2026

informative:
  JMAP-GRAPH-EDIT:
    title: "JMAP Graph Live Editing"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-graph-edit-00
    date: 2026
  JMAP-GRAPH-SEARCH:
    title: "JMAP Graph Search"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-graph-search-00
    date: 2026
  JMAP-OBJ-HISTORY:
    title: "JMAP Object History"
    author:
      - fullname: Bron Gondwana
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-object-history-02
    date: 2026-10-16
  DPV:
    title: "Data Privacy Vocabulary (DPV)"
    author:
      org: W3C Data Privacy Vocabularies and Controls Community Group
    target: https://w3id.org/dpv
  DTG-ROOMS:
    title: "Trust Tasks: rooms"
    author:
      org: Trust over IP Foundation, Decentralized Trust Graph Working Group
    target: https://github.com/trustoverip/dtgwg-trust-tasks-tf/tree/main/specs/rooms

--- abstract

This document defines JMAP Graph Provenance Records, an application profile of JMAP Graph. It lets a GraphNode be a record whose author states the basis of what it says (declared, observed, verified, or inferred), an estimate of confidence, the evidence it rests on, and how long and for what it may be relied on. Evidence is an edge to another node. A separate curation right lets users who did not write a record demote, retract, or pin it, and a Graph may set an age-based retention. The profile suits shared memory for communities and the software agents that work for them, where a reader needs to tell a person's statement from an agent's inference.

--- middle

# Introduction

A knowledge base shared by people and software agents mixes statements of very different weight: what someone says about themselves, what someone saw, what was checked against a source, and what an agent guessed. When these look the same, a reader cannot weigh them, and an agent that reads the store back cannot tell its own earlier guess from a fact.

{{JMAP-GRAPH}} gives such a store its structure: nodes with content and annotations, and typed edges between them. It assigns no meaning to either and leaves application profiles to other documents. This document is one such profile. It adds to GraphNode a small set of provenance properties, defines an edge type for evidence, adds a curation right to Graph sharing, and adds an age-based retention setting to Graph.

The vocabulary follows the record model of the Decentralized Trust Graph `rooms` Trust Tasks ({{DTG-ROOMS}}), so that records can be mapped between the two without loss of provenance.

## Design Principles

- **Provenance is the author's claim.** Every provenance property is stated by the record's author. The server stores and filters it and never evaluates it. A reader weighs it; nothing in this document makes a record authoritative.
- **Evidence is a reference, not a locator.** An evidence edge names what a record rests on. A reader does not fetch what it names in order to read the record.
- **Curation is separate from writing.** The right to change a record is not the right to judge it. Demoting, retracting, and pinning need their own right.
- **Expiry hides; it does not delete.** A record past its `validUntil` is left out of default listings and remains readable. Deletion happens only by retention or by destroying the node.
- **Standard JMAP and JMAP Graph.** All methods are those of {{RFC8620}} and {{JMAP-GRAPH}}. This document adds properties and filter conditions, not methods.

## Scope

The following are out of scope for this revision:

- Titles, descriptions, and tags. Content is the node's blob; other data uses annotations ({{JMAP-GRAPH}}, Section "Annotations").
- Signing of records. Authorship is the server-set `createdBy` and `updatedBy`. A deployment that needs records verifiable outside the server adds signatures by an extension with a capability of its own.
- Edge types other than evidence.
- Search over record content, which {{JMAP-GRAPH-SEARCH}} addresses.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Terminology from {{RFC8620}}, {{RFC9670}}, and {{JMAP-GRAPH}} is used throughout.

Record:
: A GraphNode in a Graph of an account for which this capability is advertised. Every GraphNode there is a record; a record with no provenance properties set is simply one whose author stated none.

Author:
: The Principal in a record's `createdBy`.

Curator:
: A user with `mayCurate` on the record's Graph ({{rights}}).

# Capability {#capability}

Servers supporting this specification advertise the `urn:ietf:params:jmap:graph:provenance` capability. This capability is meaningful only when `urn:ietf:params:jmap:graph` is also advertised for the account.

## Session-Level Capability Object

The value of `capabilities["urn:ietf:params:jmap:graph:provenance"]` at the session level is an empty JSON object `{}`.

## Account-Level Capability Object

The value of `accountCapabilities["urn:ietf:params:jmap:graph:provenance"]` is an empty JSON object `{}`. The limits of this document are fixed ({{limits}}).

# GraphNode Extensions {#node-extensions}

This document adds the following properties to GraphNode ({{JMAP-GRAPH}}). The first group is written by users with `mayWrite`; the second, by curators only ({{rights}}).

## Provenance Properties {#provenance}

`basis` (String|null, default: `null`):
: How the author came to the record's content. One of:

  - `declared`: the author's own statement, about themselves or as their position.
  - `observed`: something the author saw, measured, or was told directly.
  - `verified`: checked against evidence. A record with this basis SHOULD have at least one evidence edge ({{evidence}}).
  - `inferred`: a conclusion the author reached, including the output of a model.

  `null` means the author stated no basis. A writer that cannot tell which value applies, in particular software that produced the content, SHOULD use `inferred`. The server rejects any other value with an `invalidProperties` SetError naming `basis`.

`confidence` (Number|null, default: `null`):
: The author's own estimate, from 0 to 1 inclusive, that the content is correct. `null` means no estimate; it does not mean 1. The server rejects a value outside the range with an `invalidProperties` SetError naming `confidence`. Confidence values of different authors are not on a common scale ({{security}}).

`validUntil` (UTCDate|null, default: `null`):
: A time after which the author no longer stands by the record. `null` means no such time. The record is not removed when this time passes; see {{expiry}}.

`validFor` (PermittedUse|null, default: `null`):
: The uses the author intends the record for. A PermittedUse object has the following properties, at least one of which is present:

  `purposes` (String\[]|null, default: `null`):
  : Between 1 and 8 IRIs ({{RFC3987}}) naming purposes. Terms of the Data Privacy Vocabulary ({{DPV}}) are RECOMMENDED so that independent readers understand them.

  `note` (String|null, default: `null`):
  : Plain text describing the intended use, for people to read.

  The server rejects, with an `invalidProperties` SetError naming `validFor`, a PermittedUse with neither member, or with a `purposes` array that is empty or has more than 8 entries. `validFor` states the author's intent. It is not an access control and is not enforced by the server.

`credentialDigest` (String|null, default: `null`):
: On a node that stands for a credential ({{evidence}}), the base64url encoding without padding ({{RFC4648}}, Section 5) of the SHA-256 digest of the credential's exact bytes as issued. `null` otherwise. The digest is computed over the bytes, never over a canonical form, so that any holder of the same bytes computes the same value. The server rejects a value that is not 43 base64url characters with an `invalidProperties` SetError naming `credentialDigest`.

## Curation Properties {#curation}

`status` (String, default: `active`):
: One of:

  - `active`: the record stands.
  - `deprecated`: a curator has demoted the record. Its content and properties are kept. Readers SHOULD present it as demoted, not hide it.
  - `retracted`: a curator has withdrawn the record ({{retraction}}).

  The server rejects any other value with an `invalidProperties` SetError naming `status`.

`pinned` (Boolean, default: `false`):
: A curator has marked the record as one to keep in view. `pinned` is independent of `status`. A pinned record is exempt from retention ({{retention}}).

`curationReason` (String|null, default: `null`):
: Plain text, written with the most recent change of `status` or `pinned`, saying why.

## Retraction {#retraction}

A record is created with a `status` other than `retracted`; the server rejects a create with `status` `retracted` with an `invalidProperties` SetError naming `status`. When an update makes a record's `status` `retracted`, the server sets its `blobId`, `mediaType`, `uri`, `basis`, `confidence`, `validUntil`, `validFor`, and `credentialDigest` to `null`, and its `annotations` to `{}`, in the same update. An update that makes `status` `retracted` and also sets any of those properties to a value other than the one listed is rejected with an `invalidProperties` SetError naming that property. The node and its edges remain, so that records citing it as evidence still show that what they cited was withdrawn.

Retraction cannot be undone. The server rejects any update to a retracted record other than one to `pinned` or `curationReason`, or a destroy, with an `invalidProperties` SetError naming the property. A curator may change `status` from `deprecated` to `active`, but never from `retracted`. The server rejects, with a `forbidden` SetError, a `GraphEdge/set` that would create, update, or destroy an edge whose `fromId` is, or would become, a retracted record, so that what a withdrawn record cited stays as it was.

A server that also implements {{JMAP-OBJ-HISTORY}} and lists `GraphNode` in its `supportedTypes` SHOULD NOT return any prior version of a retracted record, and SHOULD permanently remove those versions when the record is retracted. Otherwise history would expose the content that retraction removed.

# Evidence {#evidence}

A record's evidence is the set of GraphEdge objects whose `fromId` is the record and whose `type` is `urn:ietf:params:jmap:graph:provenance#evidence`. The edge's `toId` names what the record rests on:

- **Another record.** The target is any GraphNode of the same Graph.
- **A resource outside the Graph,** such as a record in another Graph or a document on the web. The target is a node whose `uri` names it ({{JMAP-GRAPH}}, Section "Links Outside the Graph").
- **A credential.** The target is a node whose `uri` is the credential's identifier, if it has one, and whose `credentialDigest` is set ({{provenance}}). The digest lets a reader who holds a copy of the credential check that it is the one cited, without the Graph carrying the credential.

An evidence edge refers to its target as it is now, not as it was when the edge was written. A record that must cite a fixed version of another record cites a node that is not changed afterwards.

A record has at most 32 evidence edges ({{limits}}). Evidence is never proof: an evidence edge says what the author relied on, not that the target supports the record.

# Sharing Rights {#rights}

This document adds the following property to the GraphRights object ({{JMAP-GRAPH}}):

`mayCurate` (Boolean):
: The user may change the `status`, `pinned`, and `curationReason` properties of any record in this Graph.

`mayCurate` is not implied by `mayWrite`. The owner of the account that holds the Graph implicitly has it, as for every right.

The server rejects, with a `forbidden` SetError, a `GraphNode/set` update by a user without `mayCurate` that changes `status`, `pinned`, or `curationReason`. An update that changes only those properties requires `mayCurate` and does not require `mayWrite`; this replaces, for such updates, the rule of {{JMAP-GRAPH}} that every GraphNode update requires `mayWrite`.

Destroying a record requires `mayCurate`, in place of the `mayWrite` that {{JMAP-GRAPH}} requires; the server rejects a destroy by any other user with a `forbidden` SetError. A user who may write but not curate therefore cannot remove a record, pinned or not, by destroying it. Destruction by retention ({{retention}}) is the server's own and needs no right. On create, a user without `mayCurate` may set `status` only to `active`, `pinned` only to `false`, and `curationReason` only to `null`; the server rejects other values with a `forbidden` SetError. The provenance properties ({{provenance}}) follow `mayWrite`, like the rest of the node.

# Graph Extensions {#graph-extensions}

This document adds the following property to Graph ({{JMAP-GRAPH}}).

`retention` (Retention|null, default: `null`):
: How long the server keeps the Graph's records. `null` means records are kept until destroyed. Only a user with `mayAdmin` may change it; the server rejects a change by any other user with a `forbidden` SetError. A Retention object has the following properties:

  `recordDays` (UnsignedInt|null, default: `null`):
  : Records that are not retracted are destroyed this many days after their `updated` time. `null` means never.

  `tombstoneDays` (UnsignedInt|null, default: `null`):
  : Retracted records are destroyed this many days after their `updated` time. `null` means never.

  Each value, when not `null`, is between 1 and 36500. The server rejects others with an `invalidProperties` SetError naming `retention`.

## Retention {#retention}

The server destroys each record that retention makes eligible, as if by a `GraphNode/set` destroy, which also destroys its edges ({{JMAP-GRAPH}}). It need not do so at the exact moment of eligibility, and SHOULD do so within one day. Pinned records are never eligible. Nor is a node that has an edit log ({{JMAP-GRAPH-EDIT}}), because edit operations do not change `updated` and the clock would run out on a document still being edited. Because the clock runs from `updated`, any change to a record, including curation, restarts it.

Clients learn of destroyed records through `GraphNode/changes`, as for any destroy.

# Expiry {#expiry}

A record whose `validUntil` is earlier than the current time is expired. Clients SHOULD leave expired records out of default listings and out of what software agents retrieve by default, and SHOULD keep them available on explicit request. The server does not change or remove a record because it has expired. An author or curator who wants an expired record gone retracts or destroys it.

# Query Filters {#filters}

This document adds the following conditions to the FilterCondition of `GraphNode/query` ({{JMAP-GRAPH}}):

`basis` ((String|null)\[]):
: The record's `basis` is one of the given values. To match records with no basis, include `null` in the array.

`status` (String\[]):
: The record's `status` is one of the given values.

`pinned` (Boolean):
: The record's `pinned` equals the given value.

`validAt` (UTCDate):
: The record's `validUntil` is `null` or later than the given time. A client lists unexpired records by passing the current time.

A server that advertises this capability supports all four conditions.

# Limits {#limits}

- A record has at most 32 evidence edges. The server rejects, with an `overQuota` SetError, a `GraphEdge/set` create or update that would give a record more than 32.
- A PermittedUse has at most 8 `purposes`.

# Examples

A record written by a software agent, citing an earlier record and a credential:

~~~ json
[["GraphNode/set", {
  "accountId": "A1",
  "create": {
    "r1": {
      "graphId": "G1",
      "blobId": "Bk7q",
      "mediaType": "text/markdown",
      "basis": "inferred",
      "confidence": 0.6,
      "validUntil": "2027-01-31T00:00:00Z",
      "validFor": {
        "purposes": ["https://w3id.org/dpv#ServiceProvision"]
      }
    },
    "c1": {
      "graphId": "G1",
      "uri": "urn:uuid:4f1c9a3e-8d2b-4e7f-9a61-0c5d2b7e8f13",
      "credentialDigest": "htSyDucPJhKUVHKPpzpMNO8bl50vhWucS4VMDVuc_ds"
    }
  }
}, "0"],
["GraphEdge/set", {
  "accountId": "A1",
  "create": {
    "e1": {
      "graphId": "G1", "fromId": "#r1", "toId": "N42",
      "type": "urn:ietf:params:jmap:graph:provenance#evidence"
    },
    "e2": {
      "graphId": "G1", "fromId": "#r1", "toId": "#c1",
      "type": "urn:ietf:params:jmap:graph:provenance#evidence"
    }
  }
}, "1"]]
~~~

A curator demotes it:

~~~ json
[["GraphNode/set", {
  "accountId": "A1",
  "update": {
    "N77": {
      "status": "deprecated",
      "curationReason": "Superseded by the March inventory."
    }
  }
}, "0"]]
~~~

# Security Considerations {#security}

The considerations of {{JMAP-GRAPH}} apply. In addition:

## Provenance Is a Claim

Every provenance property is written by a user with `mayWrite` and is untrusted, like any annotation. `basis` `verified` says that the author checked, not that the check was sound. Clients SHOULD show `basis` together with the author, and SHOULD NOT present a record as more reliable than its author.

Confidence values express each author's own calibration. A client or agent SHOULD NOT rank or compare records by `confidence` across different authors, and SHOULD NOT treat a missing value as certainty.

## Records Are Data, Not Instructions

Record content, `validFor.note`, `curationReason`, and annotations are data. Software agents that read records SHOULD NOT follow instructions found in them, and SHOULD render content inert: as text, without running scripts or loading remote resources. A record store that agents read back is a channel by which anyone with `mayWrite` can try to direct those agents.

## Evidence Is Not Fetched

Clients and agents SHOULD NOT fetch the `uri` of an evidence target in order to read or weigh a record. Fetching would let a record's author make every reader contact a host the author chose, disclosing the reader's interest and address, and could reach resources the reader can access and the author cannot. The considerations of {{JMAP-GRAPH}}, Section "Links to Outside Resources", apply to a reader who follows a link deliberately.

## Credential Digests

A `credentialDigest` identifies a credential to anyone who holds it. Writing one into a Graph tells every reader of the Graph which credential the record relies on, to the extent they can obtain the credential. Authors SHOULD NOT cite a credential whose use they would not disclose to every reader of the Graph.

## Curation Authority

A curator can withdraw any record's content permanently. Owners grant `mayCurate` with that in mind. Because `mayCurate` is separate from `mayWrite`, a Graph can let many users add records and only a few judge them.

## Retention and Expiry

Retention deletes; expiry does not. A deployment that must keep records for a period, for legal or other reasons, chooses `recordDays` and `tombstoneDays` no shorter than that period. A pinned record is never removed by retention.

# IANA Considerations {#iana}

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP Capabilities" registry:

Capability Name:
: `urn:ietf:params:jmap:graph:provenance`

Intended Use:
: common

Change Controller:
: IETF

Specification document:
: This document.

Security and privacy considerations:
: See {{security}} of this document.

--- back

# Non-Normative Notes

## Mapping to DTG Rooms

The record model of the DTG `rooms` Trust Tasks ({{DTG-ROOMS}}) informed this profile. A gateway between the two can map:

| DTG rooms | This document |
|---|---|
| record `key`, `version` | GraphNode `id`, JMAP state |
| `author` | `createdBy` |
| `body`, `file` | the node's blob |
| `basis`, `confidence`, `validUntil` | the same properties, same values |
| `validFor` (`purposes`, `note`) | `validFor`, same members |
| `evidence` `kind: record` | evidence edge to the record's node |
| `evidence` `kind: credential` | evidence edge to a node with `credentialDigest` |
| `status` (`active`, `deprecated`, `retracted`), `pinned`, `reason` | `status`, `pinned`, `curationReason` |
| `curate` authority | `mayCurate` |
| `recordRetention` (`recordDays`, `tombstoneDays`) | Graph `retention` |

The mapping loses information in three places. Rooms' `title`, `description`, and `tags` have no property here and map to annotations. Rooms' credential digest is computed over a canonical form of the credential, and this document's over its exact bytes, so a gateway recomputes it and needs the credential to do so. A rooms evidence reference can name a record version; an evidence edge here cannot ({{evidence}}).

## Why Evidence Is an Edge

Evidence could have been a list property of the record. As edges, it is ordinary graph structure: a reader finds what a record rests on with one `GraphEdge/query` on `fromId`, and finds every record that rests on a given node with one on `toId`. Withdrawing a cited record is visible from every record that cites it, because retraction keeps the node ({{retraction}}).

## End-to-End Encrypted Deployments

Where the server cannot read records, as when Graph data travels end to end encrypted, the checks this document assigns to the server are made by each client against the rights it knows. Retention cannot then be applied by each client to its own copy at its own time, because copies would differ: a record one client has removed may still be updated or cited by others. Instead a client submits an ordinary destroy once a record is eligible, and every client checks eligibility against times all of them agree on, such as those of a shared ordering service. The properties, values, and rules are otherwise unchanged.
