---
title: JMAP Graph Projection
abbrev: JMAP Graph Projection
docname: draft-atwood-jmap-graph-projection-00
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
  JMAP-OBJECT-URI:
    title: "JMAP Object URIs"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-object-uri-00
    date: 2026
  RFC2119:
  RFC8174:
  RFC8620:
  RFC9404:
  RFC9670:
  JMAP-GRAPH:
    title: "JMAP for Graphs"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-graph-00
    date: 2026

informative:
  JMAP-CHAT-CALENDARS:
    title: "JMAP Chat Calendars"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-calendars-00
    date: 2026
  JMAP-CHAT-TASKS:
    title: "JMAP Chat Tasks"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-tasks-00
    date: 2026
  RFC8621:
  RFC9610:

--- abstract

This document defines JMAP Graph Projection, a companion to JMAP Graph ({{JMAP-GRAPH}}). A server MAY present the other JMAP data in an account (mail, contacts, and any other data type with a registered object URI type name) as a single Graph: each object becomes a GraphNode that refers back to it, and each reference from one object to another becomes a GraphEdge. Clients can then explore the relationships among all of an account's data with the GraphNode and GraphEdge methods, without knowing each data type in advance. The projection is filtered for each user by that user's rights on the underlying data. Whether a server accepts writes to the projection, and what a write means, is server-defined in this revision.

--- middle

# Introduction

JMAP data types refer to one another by id: an Email names its Mailboxes and its Thread, a Mailbox names its parent, an EmailSubmission names its Email and Identity, a ContactCard names its AddressBooks. Each such reference is an edge in an implicit graph. {{JMAP-GRAPH}} defines an explicit graph data model; this document defines how a server presents an account's existing data in that model.

The projection is mechanical. A node carries only a reference to its source object, and an edge's type is the name of the property that holds the reference. No relationship is given a meaning beyond the property that defines it, so the projection needs no per-type vocabulary and covers data types defined after this document without change.

## Design Principles

- **Optional.** A server MAY provide a projection. Nothing in {{JMAP-GRAPH}} or any other document depends on one existing.
- **Writes are server-defined.** A server MAY accept writes to the projection. Their meaning is server-defined in this revision ({{writes}}).
- **Nothing copied.** A projected node holds the identity of its source object, not its properties. Clients fetch content with the native `/get` method.
- **No new access.** A user sees exactly the objects, and the references between them, that the user can already read natively.

## Relationship to JMAP Graph

This document uses the Graph, GraphNode, and GraphEdge data types and methods of {{JMAP-GRAPH}} unchanged, with these differences for a projected Graph:

- Its rights, including the account owner's, and the meaning of any write to it, are server-defined ({{writes}}).
- Its contents are filtered for each user by native rights ({{visibility}}). This is an exception to the rule of {{JMAP-GRAPH}} that the Graph is the unit of access control; it is the only way to present data whose native access control is finer than a whole account.

It also adds one property to GraphNode ({{object-uri-property}}) and one GraphNode/query filter ({{query-filter}}).

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Terminology from {{RFC8620}}, {{RFC9670}}, and {{JMAP-GRAPH}} is used throughout.

Source object:
: A JMAP object, of a data type other than those defined in {{JMAP-GRAPH}}, that is represented in a projected Graph.

Projected Graph:
: The Graph that a server presents under this document for an account.

# Capability {#capability}

A server that provides a projection for an account advertises the `urn:ietf:params:jmap:graph:projection` capability for that account. This capability is meaningful only when `urn:ietf:params:jmap:graph` is also advertised for the account.

## Session-Level Capability Object

The value of `capabilities["urn:ietf:params:jmap:graph:projection"]` at the session level is an empty JSON object `{}`.

## Account-Level Capability Object

The value of `accountCapabilities["urn:ietf:params:jmap:graph:projection"]` is a JSON object with the following field:

`graphId` (Id):
: The id of the account's projected Graph.

An account has at most one projected Graph. It lives in the account whose data it projects.

# Projected Graph Contents {#contents}

## Which Objects Are Projected {#projected-types}

The server MAY project objects of any data type available in the account, subject to the following:

- A data type is projected only if it has a type name registered in the "JMAP Object URI Types" registry ({{JMAP-OBJECT-URI}}), because each node refers to its source object by object URI ({{object-uri-property}}). This document registers type names for the data types of {{RFC8621}} and {{RFC9610}} ({{iana}}).
- The server MUST NOT project the Graph, GraphNode, or GraphEdge data types. Projecting them would make the projection contain itself.
- If the server projects a data type, it projects every object of that type in the account. Which objects a given user sees is then determined by {{visibility}}.

Which data types a server projects is server-defined. Clients learn them from the object URIs of the nodes they receive.

## Nodes {#nodes}

Each source object is represented by exactly one GraphNode in the projected Graph. Its properties are:

- `graphId`: the projected Graph's id.
- `objectUri`: the source object's object URI ({{object-uri-property}}).
- `annotations`: `{}`.
- `blobId` and `mediaType`: `null`, even when the source object refers to a blob.
- `created` and `updated`: the time the node was added to the projection.
- `createdBy` and `updatedBy`: `null`.

A node's id SHOULD remain the same for as long as its source object exists, so that `GraphNode/changes` reports real changes rather than replacing nodes. When a source object is destroyed, its node is destroyed.

## Edges {#edges}

The server derives edges from the top-level properties of each projected source object. A property produces edges when its definition says its value is the id of an object of a JMAP data type, or a collection of such ids. Specifically:

- A property of type `Id` or `Id|null` produces one edge when its value is not `null`.
- A property of type `Id[]` produces one edge for each element.
- A property of type `Id[T]`, for any `T`, produces one edge for each key. The value associated with each key is not carried on the edge.

An edge is produced only when the referenced object is itself projected, in the same account. References to blobs, to objects of data types that are not projected, and to objects in other accounts produce no edge.

Each edge has the following properties:

- `graphId`: the projected Graph's id.
- `fromId`: the node of the source object that holds the property.
- `toId`: the node of the referenced object.
- `type`: the data type name of the referencing object, as registered in the "JMAP Data Types" registry ({{RFC9404}} Section 6.3), followed by `/`, followed by the property name. For example, the edge from an Email to one of its Mailboxes has type `Email/mailboxIds`. Clients match edge types by this exact string, so the server MUST construct it this way.
- `annotations`: `{}`.
- `created` and `updated`: the time the edge was added to the projection.
- `createdBy` and `updatedBy`: `null`.

There is one edge for each combination of source object, property, and referenced id. The server does not update an edge it derives: when a reference changes, the server destroys the old edge and creates a new one.

This section describes the nodes and edges the server derives. Writes may add, change, or remove nodes and edges in server-defined ways ({{writes}}).

Properties nested inside other values (for example, an id inside an object within an array) do not produce edges in this revision.

## Changes {#changes}

The projected Graph's GraphNode and GraphEdge states change whenever the projection changes, and changes are reported through the standard `/changes`, `/queryChanges`, and push mechanisms of {{RFC8620}}. A server that cannot calculate changes for a projected Graph over some interval returns `cannotCalculateChanges` ({{RFC8620}} Section 5.2), and the client resynchronizes.

# GraphNode Extensions

## objectUri {#object-uri-property}

This document adds the following property to GraphNode:

`objectUri` (String|null, server-set, default: `null`):
: The JMAP object URI ({{JMAP-OBJECT-URI}}) of the node's source object, for example `jmap:email:A1:M42`. Always `null` for nodes that are not in a projected Graph.

## GraphNode/query Filter {#query-filter}

This document adds the following property to the GraphNode/query FilterCondition defined in {{JMAP-GRAPH}}:

`objectUri` (String[]):
: The node's `objectUri` is one of these values.

This lets a client that holds native object ids find their nodes, and so start a traversal from any native object.

# Writes {#writes}

A server MAY accept creates, updates, and destroys of GraphNode and GraphEdge objects in the projected Graph, and changes to the projected Graph's `name` and `annotations`. What such a write means is server-defined in this revision. For example, a server might apply the destruction of an edge of type `Email/mailboxIds` as removing the Email from that Mailbox, keep client-written annotations as graph-only data, or accept no writes at all. Clients cannot rely on any particular effect without knowledge of the server.

The projected Graph's `myRights` are server-defined for each user, including the account owner; this overrides the rule of {{JMAP-GRAPH}} that the account owner implicitly has all rights. `mayRead` is `true` for every user who can see the projected Graph. `mayWrite` is `true` if the server accepts at least some writes from the user; a write the server does not accept is rejected with a `forbidden` SetError. `mayAdmin` is `false` and `shareWith` is `null`, because who sees what in the projection is governed by native rights ({{visibility}}), not by sharing. Each user may change their own `isSubscribed` as for any Graph.

The outcome of a write is reported through the `/set` response and through `/changes` as for any Graph. Because the server continues to derive nodes and edges from the source objects ({{contents}}), a written object need not persist as written.

# Visibility {#visibility}

The projected Graph is visible to every user who has access to the account. Its contents are filtered for each user:

- A node is visible to a user if and only if the user could fetch its source object with that data type's native `/get` method.
- An edge is visible to a user if and only if both of its nodes are visible to that user.

Servers SHOULD apply this filtering to every method on the projected Graph's nodes and edges. An invisible node or edge is reported as `notFound` by `/get` and is absent from `/query` results. When a node or edge becomes visible to a user (for example, because a Mailbox was shared with them), it is reported to that user as created by `/changes`; when it stops being visible, it is reported as destroyed.

The edge rule means that a user never learns, through the projection, that an object they cannot read exists or what refers to it.

# Example

An account holds an Email `M42` in Mailbox `MB1`, in Thread `T7`. A client finds the Email's node and the edges leaving it:

~~~json
{
  "using": [
    "urn:ietf:params:jmap:core",
    "urn:ietf:params:jmap:graph",
    "urn:ietf:params:jmap:graph:projection"
  ],
  "methodCalls": [
    ["GraphNode/query", {
      "accountId": "A1",
      "filter": {
        "graphId": "PG1",
        "objectUri": ["jmap:email:A1:M42"]
      }
    }, "0"],
    ["GraphEdge/query", {
      "accountId": "A1",
      "filter": { "graphId": "PG1", "fromId": ["N9"] }
    }, "1"],
    ["GraphEdge/get", {
      "accountId": "A1",
      "#ids": {
        "resultOf": "1",
        "name": "GraphEdge/query",
        "path": "/ids"
      },
      "properties": ["toId", "type"]
    }, "2"]
  ]
}
~~~

Here `N9` is the node id returned by the first call; a client that does not yet know it sends the edge query in a second request. The edges returned have types `Email/mailboxIds` (to the node for `MB1`) and `Email/threadId` (to the node for `T7`).

# Security Considerations {#security}

## No Widening of Access

The projection exposes nothing a user cannot already read: the visibility rules of {{visibility}} tie every node to native read access, and every edge to read access on both of its ends. A server that fails to apply them discloses the existence of objects, and the relationships between them, to users who may not read them. Because native rights can change at any time (for example, when a Mailbox is shared or unshared), servers SHOULD evaluate visibility from current native rights rather than from a cached snapshot.

## Writes to Source Objects

A write to the projection that changes a source object SHOULD be subject to the same rights checks, validation, and side effects as the corresponding native method, so that the projection does not become a path around them.

## Cost

A projection can be very large: every message in a mailbox is a node, and a Mailbox holding a million Emails is the target of a million edges. Servers MAY project only some data types, and the standard query limits of {{RFC8620}} apply. Clients exploring a projected Graph should expect high-degree nodes.

## Privacy

The projection does not reveal new information, but it makes the relationships in a user's data easy to collect in bulk. This is the same exposure as the native data, presented in a form that is easier to analyze.

# IANA Considerations {#iana}

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP Capabilities" registry:

Capability Name:
: `urn:ietf:params:jmap:graph:projection`

Intended Use:
: common

Change Controller:
: IETF

Specification document:
: This document.

Security and privacy considerations:
: See {{security}} of this document.

## JMAP Object URI Type Registrations

IANA is requested to register the following entries in the "JMAP Object URI Types" registry defined by {{JMAP-OBJECT-URI}}:

| Type Name | Data Type | Reference |
|---|---|---|
| mailbox | Mailbox | This document |
| thread | Thread | This document |
| email | Email | This document |
| identity | Identity | This document |
| emailsubmission | EmailSubmission | This document |
| addressbook | AddressBook | This document |
| contactcard | ContactCard | This document |

--- back

# Design Notes

## Why the meaning of writes is server-defined

Mapping generic graph operations onto native semantics differs by data type and has no single right answer. Destroying an edge of type `Email/mailboxIds` could mean moving the Email, removing a label, or nothing at all, and since {{RFC8621}} requires an Email to be in at least one Mailbox, some such writes cannot be applied as stated. A write that is kept only in the graph, on the other hand, stores data in a second place beside the native object. This revision lets servers choose, so that deployments can gain experience; a future document may standardize the meaning of writes for particular data types, at which point clients could rely on it.

## Why one Graph per account

Native access control is often finer than an account: per Mailbox, per AddressBook, per FileNode subtree. Projecting one Graph per such container would keep the per-Graph access rule of {{JMAP-GRAPH}}, but every reference that crosses containers (an Email in two Mailboxes, a Task that refers to a Chat) would become an edge between Graphs, which {{JMAP-GRAPH}} does not allow. One Graph per account, filtered per user, keeps those references.

## Types not registered here

This document registers object URI type names only for data types defined in published RFCs. Other documents register further type names: {{JMAP-CHAT-CALENDARS}} registers `calendarevent` and `principal`, and {{JMAP-CHAT-TASKS}} registers `task`, so those data types are projectable too. The remaining data types, including the other JMAP Calendars and JMAP Tasks data types, those of JMAP File Storage, and those of the JMAP Chat documents, become projectable when a document registers type names for them.

# Acknowledgements

The author thanks the authors of {{RFC8620}}, {{RFC8621}}, and {{RFC9610}}, whose consistent use of id references makes a mechanical projection possible.
