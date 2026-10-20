---
title: JMAP for Graphs
abbrev: JMAP Graph
docname: draft-atwood-jmap-graph-00
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
  RFC8620:
  RFC9670:

informative:
  RFC6901:
  WAVE-FED:
    title: "Google Wave Federation Protocol Over XMPP"
    author:
      - fullname: Anthony Baxter
      - fullname: Jochen Bekmann
      - fullname: Daniel Berlin
      - fullname: Joe Gregorio
      - fullname: Soren Lassen
      - fullname: Sam Thorogood
    date: 2009-07
    target: https://www.waveprotocol.org/protocol/draft-protocol-specs/draft-protocol-spec
  TINKERPOP:
    title: "Apache TinkerPop: The Property Graph"
    author:
      org: The Apache Software Foundation
    target: https://tinkerpop.apache.org/docs/current/reference/#graph-computing

--- abstract

This document defines JMAP Graph, a JMAP capability ({{RFC8620}}) for storing and synchronizing general-purpose directed graphs. It defines the `urn:ietf:params:jmap:graph` capability and three data types: Graph, a shareable container; GraphNode, a vertex with an open set of annotations and optional blob content; and GraphEdge, a directed, typed, annotated link between two nodes of the same Graph. The specification deliberately assigns no meaning to node content, edge types, or annotations, and imposes no structural constraints such as acyclicity or ordering, so that outlines, threaded conversations, mind maps, knowledge graphs, and other structures can be built on it as conventions rather than as protocol extensions.

--- middle

# Introduction

Many collaborative applications are, underneath, a graph: a threaded conversation is a tree of replies, an outline is an ordered tree of items, a mind map or knowledge base is a graph of linked notes, and a project plan is a graph of dependencies. Each of these is commonly given its own data model and its own protocol. This document instead defines one graph substrate over JMAP and leaves the interpretation of the graph to the application.

A Graph contains GraphNode and GraphEdge objects. Nodes and edges each carry an `annotations` map of arbitrary JSON values under application-chosen keys. A node may also carry a blob as content (for example, a document). An edge connects two nodes of the same Graph and carries an opaque `type` string. Synchronization, sharing, and change tracking reuse the standard mechanisms of {{RFC8620}} and {{RFC9670}}.

## Design Philosophy

- **No opinion on meaning.** This document does not define any edge type, annotation key, or node content format. Applications define their own conventions; separate documents may later standardize common ones.
- **No opinion on shape.** Cycles, self-loops, and multiple edges between the same pair of nodes are all permitted. A tree, a DAG, or an ordered list is a convention an application maintains, not a property the server enforces. As a result, the server never needs to reject a concurrent edit for producing a disallowed shape.
- **No opinion on order.** Nodes and edges have no protocol-defined order. Applications that need one (for example, sibling order in an outline) store a sort key as an annotation.
- **One integrity rule.** An edge always connects two nodes that exist in the edge's Graph. Destroying a node destroys its edges.
- **The Graph is the unit of access.** Sharing is granted per Graph. There are no per-node or per-edge access controls.
- **Standard JMAP only.** All methods are the standard methods of {{RFC8620}}. No new transport or method shape is defined.

## Scope

The following are out of scope for this revision. Each may be addressed by a separate document, and this document is written so as not to constrain their design:

- **Search.** The query filters defined here are structural only: Graph membership, edge endpoints, and edge type. They are not a search facility. Search over annotations, node content, or graph structure is left to future documents.
- **Concurrent editing within node content.** A node's content is a blob, replaced as a whole. Fine-grained real-time co-editing of a single node's content is left to a future document.
- **Low-latency transport.** Changes are announced by standard JMAP push ({{RFC8620}} Section 7). A WebSocket companion is left to a future document.
- **Federation** of Graphs between servers.
- **End-to-end encryption.** In this revision, all graph data is visible to the server.
- **Edges between Graphs.**
- **Application profiles** that assign meaning to edge types and annotation keys.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Terminology and type notation from {{RFC8620}} and {{RFC9670}} are used throughout. In particular, `String[*]` denotes a JSON object whose keys are strings and whose values may be any JSON value.

## Accounts {#account-model}

A Graph and every GraphNode and GraphEdge in it live in exactly one JMAP account: the account in which the Graph was created. Other users reach a shared Graph by accessing that account as a shared account ({{RFC8620}} Section 1.6.2), as described in {{RFC9670}}. Every `accountId` in a method call or response defined here is the id of the account that holds the Graph.

# Capability {#capability}

Servers supporting this specification MUST advertise the `urn:ietf:params:jmap:graph` capability in the JMAP Session object.

## Session-Level Capability Object

The value of `capabilities["urn:ietf:params:jmap:graph"]` at the session level is an empty JSON object `{}`.

## Account-Level Capability Object

The value of `accountCapabilities["urn:ietf:params:jmap:graph"]` is a JSON object with the following field:

`maxAnnotationSize` (UnsignedInt|null):
: The maximum size, in octets, of the `annotations` property of a single Graph, GraphNode, or GraphEdge, measured as UTF-8 encoded JSON. `null` means the server advertises no limit. A create or update that would exceed the limit is rejected with a `tooLarge` SetError ({{RFC8620}} Section 5.3).

Servers that support sharing Graphs with other users also advertise `urn:ietf:params:jmap:principals` ({{RFC9670}}).

## Principal Capability Object {#principal-capability}

When the server supports Principals ({{RFC9670}} Section 2), the `capabilities` property of a Principal may include `urn:ietf:params:jmap:graph`, whose value is a JSON object with the following fields:

`accountId` (Id|null):
: The id of the account holding the Principal's Graphs, or `null` if the user has no access to it.

`mayShareWith` (Boolean):
: `true` if the user may add this Principal to the `shareWith` property of a Graph.

# Annotations {#annotations}

Graph, GraphNode, and GraphEdge each have an `annotations` property of type `String[*]`. Annotation keys are non-empty strings; values are arbitrary JSON. This document defines no annotation keys and attaches no meaning to any value.

Keys that are meant to be understood beyond a single application SHOULD begin with a reversed domain name controlled by whoever defines the key (for example, `com.example.label`), so that independently defined keys do not collide. The same convention is recommended for GraphEdge `type` values.

Annotations are updated through the PatchObject mechanism of `/set` ({{RFC8620}} Section 5.3), which addresses a single key with the path `annotations/<key>`. Because a patch changes only the keys it names, two clients that concurrently change different keys of the same object both succeed, and the result contains both changes. When two clients change the same key, the update the server applies last determines the stored value. A client that replaces the whole `annotations` property instead discards any concurrent change to other keys; clients SHOULD patch individual keys when other users may be editing the same object.

A key that contains `/` or `~` is escaped in a patch path as described in {{RFC6901}} (`~1` and `~0` respectively). The reversed-domain convention above avoids `/` in keys for this reason.

# Data Types

## Graph {#graph}

A Graph is a container of nodes and edges, and the unit of sharing. A Graph object has the following properties:

`id` (Id, immutable, server-set):
: The id of the Graph.

`name` (String, default: `""`):
: A user-visible name for the Graph.

`annotations` (String[*], default: `{}`):
: Application data about the Graph as a whole ({{annotations}}).

`created` (UTCDate, server-set):
: When the Graph was created.

`updated` (UTCDate, server-set):
: When a property of the Graph object itself was last changed. Changes to the Graph's nodes and edges do not change this value.

`isSubscribed` (Boolean, default: `true` for the user who creates the Graph):
: As defined in {{RFC9670}} Section 4. When a Graph is shared with a user, its initial value for that user is server-defined.

`myRights` (GraphRights, server-set):
: The rights the user has on this Graph, as defined in {{RFC9670}} Section 4.

`shareWith` (Id[GraphRights]|null, default: `null`):
: As defined in {{RFC9670}} Section 4.

A GraphRights object has the following properties:

`mayRead` (Boolean):
: The user may fetch this Graph and every GraphNode and GraphEdge in it.

`mayWrite` (Boolean):
: The user may create, update, and destroy GraphNode and GraphEdge objects in this Graph, and may change the Graph's `name` and `annotations`.

`mayAdmin` (Boolean):
: The user may change the Graph's `shareWith` property and may destroy the Graph.

The owner of the account that holds the Graph implicitly has all rights.

## GraphNode {#graph-node}

A GraphNode is a vertex in a Graph. A GraphNode object has the following properties:

`id` (Id, immutable, server-set):
: The id of the node.

`graphId` (Id, immutable):
: The id of the Graph the node belongs to.

`annotations` (String[*], default: `{}`):
: Application data about the node ({{annotations}}).

`blobId` (Id|null, default: `null`):
: The id of a blob holding the node's content, or `null` if the node has no content. Content is opaque to the server.

`mediaType` (String|null, default: `null`):
: The media type of the content identified by `blobId`. When `blobId` is not `null` and `mediaType` is `null`, the content is treated as `application/octet-stream`.

`created` (UTCDate, server-set):
: When the node was created.

`updated` (UTCDate, server-set):
: When any property of the node was last changed.

`createdBy` (Id|null, server-set):
: The id of the Principal ({{RFC9670}}) that created the node, or `null` if the server does not support Principals or cannot determine it. Principal ids are found in the account given by `accountIdForPrincipal` ({{RFC9670}} Section 1.5.2).

`updatedBy` (Id|null, server-set):
: The id of the Principal that last changed the node, or `null` under the same conditions as `createdBy`.

## GraphEdge {#graph-edge}

A GraphEdge is a directed link from one node to another in the same Graph. A GraphEdge object has the following properties:

`id` (Id, immutable, server-set):
: The id of the edge.

`graphId` (Id, immutable):
: The id of the Graph the edge belongs to.

`fromId` (Id):
: The id of the GraphNode the edge starts at.

`toId` (Id):
: The id of the GraphNode the edge ends at. It may equal `fromId`.

`type` (String, default: `""`):
: An application-defined edge type. The server attaches no meaning to it. See {{annotations}} for the recommended naming convention.

`annotations` (String[*], default: `{}`):
: Application data about the edge ({{annotations}}).

`created`, `updated`, `createdBy`, `updatedBy`:
: As defined for GraphNode.

Any number of edges, of the same or different types, may connect the same pair of nodes in either direction. An edge's `fromId` and `toId` may be changed by an update; this is how an application moves a node to a new parent in a tree-shaped convention, for example.

# Methods

All methods in this section are the standard methods of {{RFC8620}} Section 5, with the additions and restrictions given here. Each data type has its own state string and is a valid type name in a StateChange object ({{RFC8620}} Section 7.1).

## Graph Methods

### Graph/get and Graph/changes

These are standard `/get` and `/changes` methods ({{RFC8620}} Sections 5.1 and 5.2).

### Graph/set

This is a standard `/set` method ({{RFC8620}} Section 5.3).

Any user with `mayRead` may change their own `isSubscribed` value. Changing `name` or `annotations` requires `mayWrite`. Changing `shareWith` is governed by {{RFC9670}} and requires `mayAdmin`. Destroying a Graph requires `mayAdmin`. A user who has `mayRead` but lacks the needed right receives a `forbidden` SetError; a user without `mayRead` receives `notFound`, as if the Graph did not exist ({{security}}).

When a Graph's `shareWith` changes, the `name` of each resulting ShareNotification ({{RFC9670}} Section 3) is the Graph's `name`.

When a Graph is destroyed, the server MUST also destroy every GraphNode and GraphEdge in it. These objects are reported as destroyed by subsequent `GraphNode/changes` and `GraphEdge/changes` calls.

### Graph/query and Graph/queryChanges

These are standard `/query` and `/queryChanges` methods ({{RFC8620}} Sections 5.5 and 5.6).

A FilterCondition for Graph/query has the following property:

`isSubscribed` (Boolean):
: The Graph's `isSubscribed` property equals this value.

The following sort properties are supported: `name`, `created`, `updated`.

## GraphNode Methods

### GraphNode/get and GraphNode/changes

These are standard `/get` and `/changes` methods.

### GraphNode/set

This is a standard `/set` method. Creating, updating, or destroying a node requires `mayWrite` on its Graph.

On create, `graphId` is required. If it does not identify a Graph the user may read, the server rejects the create with an `invalidProperties` SetError naming `graphId`. Like any id-typed property, `graphId` may reference a Graph created earlier in the same request by its creation id ({{RFC8620}} Section 5.3).

When a node is destroyed, the server MUST also destroy every GraphEdge whose `fromId` or `toId` is that node. These edges are reported as destroyed by subsequent `GraphEdge/changes` calls.

### GraphNode/query and GraphNode/queryChanges

These are standard `/query` and `/queryChanges` methods.

The `filter` argument MUST be a single FilterCondition, not a FilterOperator, and MUST include `graphId`. A server receiving any other filter, or no filter, rejects the call with an `unsupportedFilter` error ({{RFC8620}} Section 5.5). Requiring a Graph keeps every query inside a single unit of access control.

A FilterCondition for GraphNode/query has the following property:

`graphId` (Id):
: The node is in this Graph.

The following sort properties are supported: `created`, `updated`.

## GraphEdge Methods

### GraphEdge/get and GraphEdge/changes

These are standard `/get` and `/changes` methods.

### GraphEdge/set

This is a standard `/set` method. Creating, updating, or destroying an edge requires `mayWrite` on its Graph.

On create, `graphId`, `fromId`, and `toId` are required, with `graphId` validated as for GraphNode/set. On create, and on any update that changes `fromId` or `toId`, the server MUST verify that each of `fromId` and `toId` identifies an existing GraphNode in the edge's Graph. If either does not, the server MUST reject the operation with an `invalidProperties` SetError naming the failing property. The server SHOULD NOT distinguish a node that does not exist from a node in another Graph in this error.

`fromId` and `toId` may reference nodes created earlier in the same request by creation id ({{RFC8620}} Section 5.3).

Destroying an edge does not affect its nodes.

### GraphEdge/query and GraphEdge/queryChanges

These are standard `/query` and `/queryChanges` methods.

The `filter` argument is subject to the same restriction as for GraphNode/query: a single FilterCondition that includes `graphId`.

A FilterCondition for GraphEdge/query has the following properties. All properties given must match.

`graphId` (Id):
: The edge is in this Graph.

`fromId` (Id[]):
: The edge's `fromId` is one of these ids.

`toId` (Id[]):
: The edge's `toId` is one of these ids.

`type` (String):
: The edge's `type` equals this value exactly.

The following sort properties are supported: `created`, `updated`.

## Traversal {#traversal}

This document defines no traversal method. A client walks the graph one hop at a time: a GraphEdge/query with `fromId` (or `toId`) set to every node in the current frontier returns all edges leaving (or entering) that frontier, and a GraphEdge/get on the result yields the next frontier. Because `fromId` and `toId` accept an array, each hop costs one query regardless of how many nodes the frontier holds.

Within a single request, a GraphEdge/get can take its `ids` from the preceding GraphEdge/query by result reference ({{RFC8620}} Section 3.7). The next hop's filter cannot be built by result reference, because a result reference replaces a whole top-level argument and `fromId` sits inside `filter`; each further hop is therefore a further request.

# Examples

## Building a small graph in one request

This request creates a Graph, two nodes (one with Markdown content), and an edge between them, using creation ids to link the objects:

~~~json
{
  "using": [
    "urn:ietf:params:jmap:core",
    "urn:ietf:params:jmap:graph"
  ],
  "methodCalls": [
    ["Graph/set", {
      "accountId": "A1",
      "create": { "g": { "name": "Design notes" } }
    }, "0"],
    ["GraphNode/set", {
      "accountId": "A1",
      "create": {
        "a": {
          "graphId": "#g",
          "annotations": { "com.example.label": "Root" }
        },
        "b": {
          "graphId": "#g",
          "blobId": "B7f3",
          "mediaType": "text/markdown"
        }
      }
    }, "1"],
    ["GraphEdge/set", {
      "accountId": "A1",
      "create": {
        "e": {
          "graphId": "#g",
          "fromId": "#a",
          "toId": "#b",
          "type": "com.example.child"
        }
      }
    }, "2"]
  ]
}
~~~

## One hop of traversal

This request finds every edge leaving node `N1` and fetches each edge's target:

~~~json
{
  "using": [
    "urn:ietf:params:jmap:core",
    "urn:ietf:params:jmap:graph"
  ],
  "methodCalls": [
    ["GraphEdge/query", {
      "accountId": "A1",
      "filter": { "graphId": "G1", "fromId": ["N1"] }
    }, "0"],
    ["GraphEdge/get", {
      "accountId": "A1",
      "#ids": {
        "resultOf": "0",
        "name": "GraphEdge/query",
        "path": "/ids"
      },
      "properties": ["toId", "type"]
    }, "1"],
    ["GraphNode/get", {
      "accountId": "A1",
      "#ids": {
        "resultOf": "1",
        "name": "GraphEdge/get",
        "path": "/list/*/toId"
      }
    }, "2"]
  ]
}
~~~

The returned nodes are the next frontier; their ids become the `fromId` array of the next hop's query.

## Changing one annotation

This method call (one entry of a request's `methodCalls`) changes a single annotation key and leaves any other keys, including ones another client is changing concurrently, untouched:

~~~json
["GraphNode/set", {
  "accountId": "A1",
  "update": {
    "N1": { "annotations/com.example.color": "red" }
  }
}, "0"]
~~~

# Security Considerations {#security}

## Access Control

Access is granted per Graph through `shareWith` ({{RFC9670}}). Servers SHOULD evaluate the requesting user's rights on the Graph for every method call that reads or changes the Graph or its contents. A Graph the user may not read SHOULD be indistinguishable from one that does not exist: `notFound` on `/get` and on `/set` update or destroy, absence from `/query` results, and the `invalidProperties` errors described above rather than `forbidden`.

There is no finer-grained access control. An application that needs some nodes visible to fewer users than others places them in a separate Graph. Edges between Graphs are excluded from this revision partly so that an edge cannot disclose the existence of a node its reader may not see.

## Untrusted Data

Annotation values, edge types, node content, and `mediaType` are supplied by any user with `mayWrite` and are untrusted. Clients SHOULD validate any value they interpret, and SHOULD treat `mediaType` as a claim rather than a guarantee when rendering content.

## Attribution

`createdBy` and `updatedBy` are set by the server and are the only authorship information this document attests. An annotation that claims authorship is just data supplied by whoever wrote it.

## Denial of Service

Destroying a node with many edges, or a Graph with many nodes, may require substantial work. Servers MAY limit the number of nodes per Graph or edges per node and reject operations that would exceed a limit with an `overQuota` SetError ({{RFC8620}} Section 5.3). The standard query limits of {{RFC8620}} apply to GraphNode/query and GraphEdge/query; the array-valued `fromId` and `toId` filters are subject to them as well.

## Server Visibility

This revision does not define end-to-end encryption. The server can read all annotations, edge types, and graph structure, and can read node content unless the application encrypts the blobs itself. Even with encrypted blobs, the structure of a graph (which nodes are linked, how often, and by whom) can reveal sensitive information.

# IANA Considerations {#iana}

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP Capabilities" registry:

Capability Name:
: `urn:ietf:params:jmap:graph`

Intended Use:
: common

Change Controller:
: IETF

Specification document:
: This document.

Security and privacy considerations:
: See {{security}} of this document.

## JMAP Data Type Registration

IANA is requested to register the following entries in the "JMAP Data Types" registry:

Type Name: Graph
Can reference blobs: No
Can use for state change: Yes
Capability: `urn:ietf:params:jmap:graph`
Specification document: This document.

Type Name: GraphNode
Can reference blobs: Yes
Can use for state change: Yes
Capability: `urn:ietf:params:jmap:graph`
Specification document: This document.

Type Name: GraphEdge
Can reference blobs: No
Can use for state change: Yes
Capability: `urn:ietf:params:jmap:graph`
Specification document: This document.

--- back

# Design Influences and Non-Normative Notes

## Influences

- **Google Wave** ({{WAVE-FED}}) is the origin of this work. Wave's wavelet was at once the unit of access control and the unit of ordering for concurrent edits, with a tree of rich-text blips inside it. The Graph here takes the same role as the wavelet, generalized: a Wave conversation becomes one convention among many (reply edges between content-bearing nodes, with sort-key annotations), rather than the data model itself.
- **Labeled property graphs** ({{TINKERPOP}}, and databases such as Neo4j) informed the choice of first-class edges with their own ids, types, and properties. Unlike RDF triples, an edge here can be annotated, updated, and destroyed individually, and two edges of the same type between the same nodes are distinct.
- **JMAP Sharing** ({{RFC9670}}) supplied the access model unchanged; its to-do list example has the same container-and-items shape as Graph and GraphNode.

## Explicit Non-Prescriptions

The following were left to applications or future documents rather than prescribed:

- **Edge type and annotation vocabularies.** No registry is created. A future document may define common vocabularies, such as for outlines or threaded conversations.
- **Structural invariants** such as acyclicity, single parents, or connectedness.
- **Ordering.** Applications store sort keys as annotations; fractional indexing (string keys chosen to sort between their neighbors) suits concurrent insertion.
- **Search and traversal queries** beyond the structural filters in this document.
- **Content formats** for node blobs, and fine-grained concurrent editing of them.

# Acknowledgements

The author thanks the designers of Google Wave, whose wavelet model this work generalizes, and the authors of {{RFC8620}} and {{RFC9670}}, on which it is built.
