---
title: JMAP Graph Live Editing
abbrev: JMAP Graph Edit
docname: draft-atwood-jmap-graph-edit-00
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
  RFC4648:
  RFC8620:
  RFC8887:
  JMAP-GRAPH:
    title: "JMAP for Graphs"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-graph-00
    date: 2026

informative:
  JMAP-GRAPH-PROJECTION:
    title: "JMAP Graph Projection"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-graph-projection-00
    date: 2026
  RFC6838:
  JMAP-CHAT:
    title: "JMAP for Chat"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-00
    date: 2026
  JMAP-CHAT-WSS:
    title: "JMAP Chat over WebSocket"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-wss-00
    date: 2026
  JMAP-GRAPH-SEARCH:
    title: "JMAP Graph Search"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-graph-search-00
    date: 2026
  FLUID-TOB:
    title: "Total order broadcast and eventual consistency"
    author:
      org: Microsoft
    target: https://fluidframework.com/docs/concepts/tob
  AUTOMERGE-FORMAT:
    title: "Binary Document Format"
    author:
      org: Automerge contributors
    target: https://automerge.org/automerge-binary-format-spec/
  YJS:
    title: "Yjs"
    author:
      org: Yjs contributors
    target: https://github.com/yjs/yjs
  WAVE-OT:
    title: "Google Wave Operational Transformation"
    author:
      - fullname: David Wang
      - fullname: Alex Mah
      - fullname: Soren Lassen
    date: 2010-07
    target: https://svn.apache.org/repos/asf/incubator/wave/trunk/whitepapers/operational-transform/operational-transform.html?p=1400000

--- abstract

This document defines JMAP Graph Live Editing, a companion to JMAP Graph ({{JMAP-GRAPH}}) that lets several users edit the content of one GraphNode at the same time, character by character, and replay its editing history. The server assigns a total order to edit operations and stores them, but never reads, transforms, or merges them; clients merge them using an edit format named by the node. This keeps the server independent of any one format and allows the operations to be encrypted end to end. The document defines a GraphNodeOp data type, extensions to GraphNode for format, snapshots, and history, and a WebSocket stream of operations and cursor positions.

--- middle

# Introduction

{{JMAP-GRAPH}} gives a GraphNode content as a blob that is replaced as a whole. That suits content one user changes at a time, but not a document that several users type into at once, as in Google Wave ({{WAVE-OT}}) or a shared editor. This document adds a log of edit operations ("ops") to a node.

## Design Principles

- **The server orders; clients merge.** The server gives each op a sequence number and stores it. It never interprets op content. Every client applies the same ops in the same order, using the node's edit format, and so reaches the same document. This is the model of total order broadcast used by the Fluid Framework ({{FLUID-TOB}}).
- **No mandatory edit format.** The node names its edit format as a media type. This revision defines the envelope that carries ops, not any format. Formats whose merge is commutative, such as the CRDTs Automerge ({{AUTOMERGE-FORMAT}}) and Yjs ({{YJS}}), fit directly ({{bindings}}).
- **Standard JMAP.** Ops are a JMAP data type with the standard methods of {{RFC8620}}. Submitting an op is a `/set` create, and its response is the acknowledgment.
- **Snapshots are the node's blob.** A client periodically writes the merged document as the node's `blobId`, recording which ops it includes. Ops covered by a snapshot may then be discarded.
- **History is a server policy.** Full playback of a node's history is possible until the server first compacts its log; how long that is depends on the server's retention.

## Scope

The following are out of scope for this revision:

- A mandatory-to-implement edit format. Two conforming clients interoperate only if they support the node's format.
- End-to-end encryption of ops. This document does not prevent it: op data and cursor data are opaque to the server.
- Signing of ops and detection of forked histories.
- Federation of edit logs between servers.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Terminology from {{RFC8620}}, {{RFC8887}}, and {{JMAP-GRAPH}} is used throughout.

Op:
: One edit operation on a node's content, encoded in the node's edit format.

Edit log:
: The sequence of ops of one node, in the order the server assigned.

Head:
: The highest sequence number assigned in a node's edit log.

# Capability {#capability}

Servers supporting this specification advertise the `urn:ietf:params:jmap:graph:edit` capability. This capability is meaningful only when `urn:ietf:params:jmap:graph` is also advertised for the account.

## Session-Level Capability Object

The value of `capabilities["urn:ietf:params:jmap:graph:edit"]` at the session level is an empty JSON object `{}`.

## Account-Level Capability Object

The value of `accountCapabilities["urn:ietf:params:jmap:graph:edit"]` is a JSON object with the following fields:

`maxOpSize` (UnsignedInt):
: The largest op, in octets before base64url encoding, that the server accepts.

`maxCursorSize` (UnsignedInt):
: The largest cursor data, in octets before base64url encoding, that the server accepts in `GraphNode/cursor` ({{cursor-method}}).

`opRetention` (UnsignedInt|null):
: The number of seconds for which the server keeps an op after it is sequenced, even when a snapshot covers it ({{compaction}}). `null` means the server promises no retention beyond what {{compaction}} requires. Because compaction removes the oldest ops first, full playback ({{compaction}}) remains possible for at least this long after a node's first op.

The WebSocket stream ({{stream}}) is available on any connection established under {{RFC8887}} to a server that advertises this capability.

# GraphNode Extensions {#node-extensions}

This document adds the following properties to GraphNode ({{JMAP-GRAPH}}).

`editFormat` (String|null, default: `null`):
: The media type ({{RFC6838}}) of the node's ops, or `null` if the node has no edit log. It may be set on create, or changed by update from `null` to a non-null value; the server rejects any other change with an `invalidProperties` SetError naming `editFormat`. Once set, it never changes, so two clients cannot give a node different formats.

`snapshotSeq` (UnsignedInt|null, default: `null`):
: The `seq` of the last op included in the node's current blob, or `null` if the node has no snapshot. See {{snapshots}}.

While `editFormat` is `null`, `snapshotSeq` is `null`; the server rejects any other value with an `invalidProperties` SetError naming `snapshotSeq`. When `editFormat` is not `null`, the node's blob is either absent or a snapshot ({{snapshots}}): `blobId` is `null` exactly when `snapshotSeq` is `null`. The server rejects, with an `invalidProperties` SetError naming `editFormat`, a create or update that would give a node a non-null `editFormat` while it has a non-null `blobId` and a `null` `snapshotSeq`. A client that converts existing content into an edited document creates that content as ops, and may then write a snapshot.

A node of a projected Graph ({{JMAP-GRAPH-PROJECTION}}) always has `editFormat` `null`, because its visibility is filtered per user and it cannot hold a snapshot. The server rejects a `GraphNode/set` that sets `editFormat` on such a node with a `forbidden` SetError.

The server tracks the head of each node's edit log, and the lowest `seq` it still holds, internally. They are not GraphNode properties, so creating an op or compacting the log does not change the GraphNode, its state string, or its `updated` and `updatedBy` properties.

# The GraphNodeOp Data Type {#graph-node-op}

A GraphNodeOp is one op in a node's edit log. Ops are immutable once created. A GraphNodeOp object has the following properties:

`id` (Id, immutable, server-set):
: The id of the op.

`graphId` (Id, immutable, server-set):
: The id of the Graph of the op's node.

`nodeId` (Id, immutable):
: The id of the GraphNode whose content the op edits.

`seq` (UnsignedInt, immutable, server-set):
: The op's position in the node's edit log. The first op of a node has `seq` 1, and each later op of that node has the previous op's `seq` plus one, with no gaps.

`baseSeq` (UnsignedInt, immutable):
: The `seq` of the last op the author had applied when creating this op, or 0 if none. Formats that merge commutatively can ignore it; formats that transform positions need it.

`data` (String, immutable):
: The op's content, encoded with base64url without padding ({{RFC4648, Section 5}}). Its meaning is defined by the node's `editFormat`.

`created` (UTCDate, immutable, server-set):
: When the op was sequenced.

`createdBy` (Id|null, immutable, server-set):
: The id of the Principal that created the op, as defined for GraphNode `createdBy` in {{JMAP-GRAPH}}.

Ops are kept in the account of their Graph and are subject to its access control ({{access}}).

# Methods {#methods}

## GraphNodeOp/get and GraphNodeOp/changes

These are standard `/get` and `/changes` methods ({{RFC8620}} Sections 5.1 and 5.2). Both require `mayRead` on the op's Graph.

## GraphNodeOp/set {#op-set}

This is a standard `/set` method ({{RFC8620}} Section 5.3), restricted to create. A create requires `mayWrite` on the node's Graph. An update or destroy is rejected with a `forbidden` SetError: ops change only by being sequenced and leave only through compaction ({{compaction}}) or when their node is destroyed.

On create, `nodeId`, `baseSeq`, and `data` are required. The server rejects the create with an `invalidProperties` SetError if `nodeId` does not identify a node the user may read whose `editFormat` is not `null` (naming `nodeId`), if `baseSeq` is greater than the head of the node's edit log (naming `baseSeq`), or if `data` is not valid base64url (naming `data`). It rejects an op whose decoded `data` exceeds `maxOpSize` with a `tooLarge` SetError.

The server MUST NOT reject an op because its `baseSeq` is old. It cannot judge whether an op still applies, because it does not read ops; the edit format decides ({{format-requirements}}).

For each accepted create, the server assigns the next `seq` of the node and returns the op's server-set properties in the `created` response ({{RFC8620}} Section 5.3). That response is the author's acknowledgment and tells it where its op sits in the order.

Creates within one `/set` call are sequenced in an order the server chooses, because the keys of the `create` argument are unordered. A client that needs two of its ops sequenced in a particular order places them in separate `GraphNodeOp/set` calls, in that order; method calls in a request are executed in order ({{RFC8620, Section 3.10}}).

The server does not detect a create that is a retry of one it already accepted. A client that retries a create whose outcome it does not know MUST resend the same `baseSeq` and `data` unchanged; the edit format then treats the second copy as having no effect ({{format-requirements}}). An op with different bytes is a different edit.

## GraphNodeOp/query and GraphNodeOp/queryChanges {#op-query}

These are standard `/query` and `/queryChanges` methods ({{RFC8620}} Sections 5.5 and 5.6). They require `mayRead` on the Graph.

The `filter` argument MUST be a single FilterCondition, not a FilterOperator, and MUST include `nodeId`; a server receiving any other filter, or no filter, rejects the call with an `unsupportedFilter` error ({{RFC8620}} Section 5.5). A FilterCondition has the following properties:

`nodeId` (Id):
: The op's node.

`afterSeq` (UnsignedInt):
: The op's `seq` is greater than this value.

The only supported sort is `seq` ascending, which is also the order when `sort` is absent.

A client catches up on a node by querying with `afterSeq` set to the last `seq` it has applied, and fetching the result with `GraphNodeOp/get`, using a result reference to the query's `ids` ({{RFC8620}} Section 3.7). A client pages a long result by sending the next query with `afterSeq` set to the last `seq` it received, not by using `position`, because compaction can remove ops from the front of the log between queries.

If the server has destroyed any op of the node whose `seq` is greater than `afterSeq` ({{compaction}}), it MUST reject the query with an `opsCompacted` error ({{errors}}). Otherwise a client would apply later ops without the missing ones and silently diverge. A client that receives this error reloads the node from its current snapshot ({{snapshots}}).

# Snapshots {#snapshots}

A snapshot is the result of applying a node's ops, from the start of its log through some `seq`, encoded as a blob. It lets a client load a node without replaying its whole history.

A client writes a snapshot with `GraphNode/set`, setting `blobId`, `mediaType`, and `snapshotSeq` together. A snapshot with `snapshotSeq` n MUST encode exactly the document produced by applying ops 1 through n, in `seq` order, to an empty document (or the ops after an earlier snapshot's `snapshotSeq` through n, to that snapshot). Its writer therefore holds every op with `seq` at most n, and includes no edit that has not been sequenced at or below n, such as its own ops whose creation has not yet succeeded. Otherwise clients that load the snapshot and clients that applied the log would hold different documents. Typically `mediaType` is the node's `editFormat`; a format may define a separate document encoding for snapshots ({{format-requirements}}). Writing a snapshot requires `mayWrite`, as for any change to a node.

For a node whose `editFormat` is not `null`:

- A `GraphNode/set` update changes `blobId` if and only if it sets `snapshotSeq`. Otherwise the server rejects it with an `invalidProperties` SetError naming the property that is missing. This prevents a client that does not know about the edit log from replacing the document and discarding concurrent edits.
- The server MUST reject, with an `invalidProperties` SetError naming `snapshotSeq`, a value of 0, a value greater than the head of the edit log, a value less than the current `snapshotSeq`, or `null` when the current `snapshotSeq` is not `null`. Because the value cannot decrease or be removed, two clients writing snapshots concurrently cannot move the node back to an older one, and a snapshot cannot be discarded once written.

To load a node, a client fetches its blob and then the ops after `snapshotSeq`, and applies them in `seq` order; if `snapshotSeq` is `null`, it starts from an empty document and applies all ops. If the query fails with `opsCompacted`, the snapshot it fetched has been superseded; it fetches the node again.

# Compaction and History {#compaction}

The server MAY destroy ops whose `seq` is less than or equal to the node's `snapshotSeq`, but only from the start of the log: it destroys an op only if it has destroyed every op of the node with a lower `seq`. The server therefore always holds a contiguous range of ops ending at the head. Destroyed ops are reported by `GraphNodeOp/changes`. The server MUST NOT destroy an op whose `seq` is greater than `snapshotSeq`, because a client loading the node needs every op after the snapshot. When `opRetention` is not `null`, the server keeps each op for at least that many seconds after its `created` time.

A client can play back a node's full history, by replaying its ops in `seq` order from an empty document, only while no op of the node has been compacted; a query with `afterSeq` 0 then succeeds. Once ops have been compacted, the remaining ops can be applied only to a document that already holds every earlier op, such as a copy a client kept locally.

When a node is destroyed, its ops are destroyed with it; when a Graph is destroyed, so are the ops of all its nodes.

# Errors {#errors}

This document defines one method-level error:

`opsCompacted`:
: Returned by `GraphNodeOp/query` when the server has destroyed an op of the node whose `seq` is greater than `afterSeq`, so that the result would not be contiguous with `afterSeq` ({{op-query}}). The client reloads the node from its snapshot.

# Edit Formats {#formats}

## Requirements on Formats {#format-requirements}

A media type is usable as an `editFormat` only if its definition specifies the following. These requirements are what make the envelope interoperable: without them, two clients applying the same log could reach different documents.

- **Determinism.** The document is a function of the ops applied, in `seq` order, starting from an empty document. Every client that applies the same ops in the same order reaches the same document.
- **Validity.** Whether an op is valid is determined only from its bytes and the ops before it in the log. Every client skips an invalid op in the same way, so a malformed or malicious op cannot make clients diverge. An op that the format defers until a later op supplies something it depends on is not invalid.
- **Duplicates.** Two ops of a node with the same `baseSeq` and identical `data` are the same edit, and applying the second has no effect, because the server does not detect retried creates ({{op-set}}). The format ensures that distinct edits never have identical `data`, for example by including an author identifier and a counter in each op.
- **Concurrency.** An op whose `baseSeq` is lower than the `seq` of the op before it is applied correctly; that is, the format merges concurrent edits.
- **Snapshots.** The definition states how a snapshot is encoded, and that loading a snapshot of ops 1 through n and then applying ops after n gives the same document as applying all ops from an empty document. Ops among 1 through n that are still deferred are part of the snapshot.

Formats that merge commutatively and identify each op uniquely, as CRDTs do, meet the duplicate and concurrency requirements through their op identifiers, without using `baseSeq`.

## Informative Bindings {#bindings}

This section is not normative. It describes how two existing CRDT libraries map onto this document. Neither has a registered media type at the time of writing, and neither has a complete normative specification, so this document names no media type for them.

- **Automerge** ({{AUTOMERGE-FORMAT}}): each op's `data` holds one encoded change chunk. A snapshot is an encoded document chunk, followed by a change chunk for each sequenced change still waiting for its dependencies. Automerge drops duplicate changes and merges concurrent ones, so it ignores `baseSeq`. To meet the exactness rule of {{snapshots}}, a client saves the document as of the sequenced changes, not its live document, which may hold changes not yet sequenced.
- **Yjs** ({{YJS}}): each op's `data` holds one update in the version 1 update encoding; a snapshot is the state encoded as a single update, which includes pending updates. Yjs merges updates in any order and ignores duplicates, so it ignores `baseSeq`. To meet the exactness rule of {{snapshots}}, a client keeps a separate document to which only sequenced updates are applied, and encodes that document.

# WebSocket Stream {#stream}

On a WebSocket connection ({{RFC8887}}), a client can subscribe to the ops of selected nodes as they are sequenced, and to the cursor positions of other users editing them. Ops sent this way are the same ops available through `GraphNodeOp/get`; the stream only delivers them sooner. Standard `StateChange` notifications for GraphNodeOp continue as usual.

A client sends ops over the WebSocket connection as ordinary JMAP requests containing `GraphNodeOp/set` calls ({{RFC8887}} Section 4.3). No other upstream message is defined for ops.

## GraphEditStreamEnable {#stream-enable}

A client sends a `GraphEditStreamEnable` message to start or replace its subscription on the current connection. It has the following properties:

`@type` (String):
: `"GraphEditStreamEnable"`.

`nodeIds` (Id[]):
: The GraphNodes whose ops and cursors the client wishes to receive. An id matches that id in every account the user can access; events carry `accountId` to tell accounts apart.

`dataTypes` (String[]):
: A non-empty list of the event categories wanted: `"ops"` for `GraphNodeOpEvent` and `"cursors"` for `GraphCursorEvent`.

`id` (String, optional):
: A client-specified identifier. A `RequestError` caused by this message carries it as `requestId` ({{RFC8887}} Section 4.3.4).

A new `GraphEditStreamEnable` replaces the previous subscription in its entirety. Success is silent; there is no acknowledgment. The subscription takes effect before the server processes any later message on the same connection. A client that starts editing a node therefore sends `GraphEditStreamEnable` first and then loads or catches up on the node ({{op-query}}), so that every op reaches it through the stream, the query, or both. If `dataTypes` is empty or contains no recognized value, the server responds with a `RequestError` whose `type` is `urn:ietf:params:jmap:error:invalidArguments` and leaves the current subscription unchanged. Unrecognized values alongside recognized ones are ignored. Ids in `nodeIds` that do not identify a node the user may read produce no events but remain in the subscription; rights are evaluated at each delivery ({{access}}), so events begin, or resume after a revocation, once the user has `mayRead`. Such ids cause no error, so the server does not reveal whether the nodes exist.

## GraphEditStreamDisable {#stream-disable}

A `GraphEditStreamDisable` message, whose only property is `@type` with the value `"GraphEditStreamDisable"`, ends the subscription. It succeeds silently whether or not a subscription is active, and does not affect any other subscription on the connection.

## GraphNodeOpEvent {#op-event}

The server sends a `GraphNodeOpEvent` for each op sequenced on a subscribed node, when `"ops"` is in `dataTypes`. It has the following properties:

`@type` (String):
: `"GraphNodeOpEvent"`.

`accountId` (Id):
: The account holding the op.

`op` (GraphNodeOp):
: The op, with all its properties.

The server SHOULD send a node's op events in `seq` order and includes the author's own ops, so that every client sees one sequence. Clients rely on `seq`, not on arrival order. For each node, a client tracks the highest `seq` H through which it has applied every op; loading a snapshot sets H to `snapshotSeq`. It discards an op event whose `seq` is at most H, applies one whose `seq` is H + 1 and increments H, and on a higher `seq` fetches the ops after H with `GraphNodeOp/query` ({{op-query}}). The `seq` returned for a client's own op by `/set` does not advance H; the op reaches the client through the stream or a query like any other, and applying it is harmless. Events are not stored for a disconnected client; after reconnecting, a client sends `GraphEditStreamEnable` again and catches up the same way.

The server delivers an op event only to users who have `mayRead` on the node's Graph at the time of delivery.

## Cursors {#cursors}

Cursor positions and selections are ephemeral. They are not stored, are best-effort, and carry no state.

### GraphNode/cursor {#cursor-method}

This method reports the calling user's cursor or selection in a node to other users subscribed to it. It takes the following arguments:

`accountId` (Id):
: The account holding the node.

`nodeId` (Id):
: The node.

`data` (String):
: The cursor or selection, encoded with base64url without padding ({{RFC4648, Section 5}}). Its meaning is defined by the node's `editFormat`.

The response has the arguments `accountId` and `nodeId` only. The method requires `mayRead` on the node's Graph. It fails with an `invalidArguments` error ({{RFC8620}} Section 3.6.2) if `nodeId` does not identify a node the user may read whose `editFormat` is not `null`, if `data` is not valid base64url, or if its decoded size exceeds `maxCursorSize`; one error covers missing and unreadable nodes, so the response does not reveal whether a node exists. Servers SHOULD rate-limit the method per user per node, and MAY discard calls above the limit while still returning success.

### GraphCursorEvent {#cursor-event}

When a user calls `GraphNode/cursor`, the server sends a `GraphCursorEvent` to other users' connections subscribed to the node with `"cursors"` in `dataTypes`. It has the following properties:

`@type` (String):
: `"GraphCursorEvent"`.

`accountId` (Id):
: The account holding the node.

`nodeId` (Id):
: The node.

`senderId` (Id|null):
: The id of the Principal whose cursor this is, or `null` if the server does not support Principals.

`data` (String):
: The cursor data as given to `GraphNode/cursor`.

The server delivers a cursor event only to users who have `mayRead` on the node's Graph at the time of delivery, and SHOULD NOT deliver it to the sender's own connections. Servers SHOULD rate-limit cursor events per sender per node and MAY drop events above the limit.

When {{JMAP-CHAT}} is also available, a server SHOULD NOT deliver a cursor event to a user who has blocked the sender, following the treatment of typing events in {{JMAP-CHAT-WSS}}. This applies only to cursor events. Ops are shared document state: dropping a blocked user's ops for one reader would make that reader's document diverge, so they are always delivered. A client may choose how to present a blocked user's edits.

# Examples

## Submitting an op

~~~json
["GraphNodeOp/set", {
  "accountId": "A1",
  "create": {
    "o1": {
      "nodeId": "N1",
      "baseSeq": 41,
      "data": "AAECAwQF"
    }
  }
}, "0"]
~~~

The response acknowledges the op and gives its place in the log:

~~~json
["GraphNodeOp/set", {
  "accountId": "A1",
  "oldState": "s41",
  "newState": "s42",
  "created": {
    "o1": {
      "id": "P42",
      "graphId": "G1",
      "seq": 42,
      "created": "2026-10-20T17:00:00Z",
      "createdBy": "U7"
    }
  }
}, "0"]
~~~

## Loading a node

A client reads the node and the ops after its snapshot in one request:

~~~json
{
  "using": [
    "urn:ietf:params:jmap:core",
    "urn:ietf:params:jmap:graph",
    "urn:ietf:params:jmap:graph:edit"
  ],
  "methodCalls": [
    ["GraphNode/get", {
      "accountId": "A1",
      "ids": ["N1"],
      "properties": ["blobId", "mediaType", "editFormat",
                     "snapshotSeq"]
    }, "0"],
    ["GraphNodeOp/query", {
      "accountId": "A1",
      "filter": { "nodeId": "N1", "afterSeq": 40 }
    }, "1"],
    ["GraphNodeOp/get", {
      "accountId": "A1",
      "#ids": {
        "resultOf": "1",
        "name": "GraphNodeOp/query",
        "path": "/ids"
      }
    }, "2"]
  ]
}
~~~

Here the client already knew that `snapshotSeq` was 40; a client that does not know it issues the query in a second request. Before loading, the client has sent `GraphEditStreamEnable` for the node ({{stream-enable}}).

## Subscribing to a node

~~~json
{
  "@type": "GraphEditStreamEnable",
  "nodeIds": ["N1"],
  "dataTypes": ["ops", "cursors"],
  "id": "sub1"
}
~~~

# Security Considerations {#security}

## Access Control {#access}

Access follows the node's Graph ({{JMAP-GRAPH}}). Reading ops, receiving op and cursor events, and calling `GraphNode/cursor` require `mayRead`; creating ops and writing snapshots require `mayWrite`. Servers SHOULD evaluate these rights at the time of each call and each event delivery, so that a user whose access is revoked stops receiving events at once. Revoking access does not remove ops the user created earlier.

## Opaque Content

The server does not read ops, so it cannot validate them. A buggy or malicious writer can append ops that other clients cannot apply. The validity requirement on formats ({{format-requirements}}) ensures that all clients skip such ops identically, so they cannot cause divergence, but they still consume log space; servers SHOULD rate-limit op creation and MAY apply quotas, rejecting creates with `overQuota` or `rateLimit` SetErrors ({{RFC8620}} Section 5.3).

A snapshot is written by a client and is no more trustworthy than any other content a user with `mayWrite` supplies. A client that doubts a snapshot can rebuild the document by replaying the log, when the server still holds it from the start.

## Attribution

`createdBy` on each op is set by the server and is the only authorship this document attests. Edit formats commonly embed their own author or actor identifiers in op data; those are claims made by the client. Clients SHOULD treat `createdBy` as authoritative when presenting who made an edit, for example during playback, and SHOULD check embedded identifiers against it where the format allows.

## History and Erasure

Text deleted from a document remains in the edit log until the ops that inserted it are compacted, which requires a snapshot that covers them ({{compaction}}). Deployments that must erase content on request write a snapshot and compact the log, and should take `opRetention` into account.

## Server Visibility

Even when op data and cursor data are encrypted by clients, the server sees which user edits which node, when, and how large each op is. Op timing can reveal typing rhythm. Deployments that encrypt content should consider this exposure.

## Search and Projection

Search ({{JMAP-GRAPH-SEARCH}}) and other server features see only the node's snapshot blob, not unsnapshotted ops, and see nothing useful if the snapshot is encrypted or in a format the server does not understand.

# IANA Considerations {#iana}

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP Capabilities" registry:

Capability Name:
: `urn:ietf:params:jmap:graph:edit`

Intended Use:
: common

Change Controller:
: IETF

Specification document:
: This document.

Security and privacy considerations:
: See {{security}} of this document.

## JMAP Data Type Registration

IANA is requested to register the following entry in the "JMAP Data Types" registry:

Type Name: GraphNodeOp
Can reference blobs: No
Can use for state change: Yes
Capability: `urn:ietf:params:jmap:graph:edit`
Specification document: This document.

## JMAP Error Code Registration

IANA is requested to register the following entry in the "JMAP Error Codes" registry ({{RFC8620}} Section 9.5):

JMAP Error Code:
: `opsCompacted`

Intended Use:
: common

Change Controller:
: IETF

Reference:
: This document, {{errors}}

Description:
: The ops requested have been compacted; the client reloads the node from its snapshot.

--- back

# Design Notes

## Why the server never merges

A server that transforms ops, as Google Wave's did ({{WAVE-OT}}), must understand the document format, which ties the protocol to one format and rules out encrypting the content. A server that only orders ops works with any format that meets {{format-requirements}}, and its job reduces to assigning sequence numbers and storing data.

## Why no format is mandatory

At the time of writing, no collaborative-editing format has both a complete normative specification and several independent implementations. The Automerge binary format has a written specification but defers to its reference implementation, and Yjs is defined by its implementation. Defining a new rich-text merge format is substantial work in its own right. This revision therefore fixes the envelope so that a later document can make a format mandatory without changing it.

## Why the server accepts stale ops

A client that was offline may submit ops whose `baseSeq` is far behind the head. Rejecting them would lose the user's work, and the server cannot tell whether they still apply. The format decides, as the concurrency requirement of {{format-requirements}} demands.

# Acknowledgements

The author thanks the designers of Google Wave, the Fluid Framework, Automerge, and Yjs, whose work this design draws on.
