---
title: JMAP Graph Search
abbrev: JMAP Graph Search
docname: draft-atwood-jmap-graph-search-00
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
  RFC6901:
  RFC8126:
  JMAP-GRAPH:
    title: "JMAP for Graphs"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-graph-00
    date: 2026

informative:
  RFC8621:
  JMAP-GRAPH-PROJECTION:
    title: "JMAP Graph Projection"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-graph-projection-00
    date: 2026
  GQL:
    title: "Information technology - Database languages - GQL"
    author:
      org: ISO/IEC
    seriesinfo:
      ISO/IEC: 39075:2024
    date: 2024-04
  OPENCYPHER:
    title: "Cypher Query Language Reference, Version 9"
    author:
      org: openCypher Implementers Group
    target: https://s3.amazonaws.com/artifacts.opencypher.org/openCypher9.pdf
  GREMLIN:
    title: "Apache TinkerPop 3.7.7 Reference Documentation"
    author:
      org: The Apache Software Foundation
    date: 2026-09
    target: https://tinkerpop.apache.org/docs/3.7.7/reference/
  SPARQL11:
    title: "SPARQL 1.1 Query Language"
    author:
      - fullname: Steve Harris
        role: editor
      - fullname: Andy Seaborne
        role: editor
    seriesinfo:
      W3C: REC-sparql11-query-20130321
    date: 2013-03-21
    target: https://www.w3.org/TR/2013/REC-sparql11-query-20130321/
  SPARQL11-FED:
    title: "SPARQL 1.1 Federated Query"
    author:
      - fullname: Eric Prud'hommeaux
        role: editor
      - fullname: Carlos Buil Aranda
        role: editor
    seriesinfo:
      W3C: REC-sparql11-federated-query-20130321
    date: 2013-03-21
    target: https://www.w3.org/TR/2013/REC-sparql11-federated-query-20130321/
  DOCUMENTDB:
    title: "DocumentDB 1.0"
    author:
      org: The Linux Foundation
    date: 2026-10-19
    target: https://github.com/documentdb/documentdb
  MONGODB-GRAPHLOOKUP:
    title: "$graphLookup (aggregation stage)"
    author:
      org: MongoDB, Inc.
    target: https://www.mongodb.com/docs/manual/reference/operator/aggregation/graphLookup/

--- abstract

This document defines JMAP Graph Search, a companion to JMAP Graph ({{JMAP-GRAPH}}) that adds search to GraphNode/query and GraphEdge/query. It defines a small built-in filter language, written as JSON in the style of document-database query filters, that every implementation supports; it covers comparisons on any property, including annotations, as well as bounded graph traversal and optional full-text matching. It also lets a server advertise further query languages, such as ISO GQL, openCypher, Gremlin, SPARQL, and the DocumentDB query language, from a new IANA registry, and defines how a Graph is presented to them. No new methods are defined; results are ordinary query results.

--- middle

# Introduction

{{JMAP-GRAPH}} defines only structural query filters: membership in a Graph, edge endpoints, and edge type. It leaves search to other documents. This document fills that gap without choosing a single search language for everyone:

- A **built-in filter language** ({{where}}) gives every server and client a common, cheap-to-implement baseline.
- **Advertised languages** ({{languages}}) let a server offer established graph or document query languages that it already supports, with no language other than the built-in one required.

## Design Principles

- **No new methods.** Both mechanisms are new properties of the GraphNode/query and GraphEdge/query FilterCondition. Paging, sorting, `/queryChanges`, and result references to `/get` work as for any JMAP query ({{RFC8620}} Sections 5.5 and 5.6).
- **Results are objects of the queried type.** A search on GraphNode/query yields GraphNodes, and on GraphEdge/query yields GraphEdges. Languages that can return paths or tables are restricted accordingly, so no new result format is needed.
- **One Graph at a time.** The rule of {{JMAP-GRAPH}} that every filter names a single Graph still applies, so search never crosses a unit of access control.
- **Read-only.** A search never changes data.
- **Bounded.** Traversal has an advertised depth limit, and servers may reject any query they consider too expensive.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Terminology from {{RFC8620}} and {{JMAP-GRAPH}} is used throughout.

Candidate:
: An object being tested against a filter: a GraphNode during GraphNode/query, or a GraphEdge during GraphEdge/query, or a node or edge reached while evaluating a traversal operator.

# Capability {#capability}

Servers supporting this specification advertise the `urn:ietf:params:jmap:graph:search` capability. This capability is meaningful only when `urn:ietf:params:jmap:graph` is also advertised for the account.

## Session-Level Capability Object

The value of `capabilities["urn:ietf:params:jmap:graph:search"]` at the session level is an empty JSON object `{}`.

## Account-Level Capability Object

The value of `accountCapabilities["urn:ietf:params:jmap:graph:search"]` is a JSON object with the following fields:

`languages` (String[]):
: The identifiers, from the "JMAP Graph Query Languages" registry ({{iana-languages}}), of the additional query languages the server supports on this account. May be empty. Clients ignore identifiers they do not recognize.

`maxDepth` (UnsignedInt):
: The largest traversal depth ({{depth}}) the server accepts in a built-in filter expression.

`supportsText` (Boolean):
: `true` if the server supports the `$text` operator ({{text}}).

# FilterCondition Extensions {#filter}

This document adds the following properties to the FilterCondition of GraphNode/query and of GraphEdge/query ({{JMAP-GRAPH}}). Like all FilterCondition properties, each one present must match, and they combine with the structural properties of {{JMAP-GRAPH}} (which still require `graphId`).

`where` (Expression):
: A built-in filter expression ({{where}}). The candidate matches if the expression is true for it.

`language` (String):
: An identifier from the account's `languages` list. Present if and only if `expression` is present.

`expression` (String):
: A query in the language named by `language`, interpreted as described in {{languages}}. The candidate matches if the query's result includes it.

A FilterCondition may include `where`, `language` with `expression`, or both.

An expression is well-formed if: every member name is a JSON Pointer ({{RFC6901}}) used as a field test or an operator defined in {{where}}; every operator object contains only `$`-prefixed names; every operand has the type stated for it; EdgeTest and ReachTest objects contain only the members defined for them and include every required member; every `direction` is `"out"`, `"in"`, or `"any"`; every ReachTest `maxDepth` is at least 1; and structural operators appear only on the kind of candidate stated in {{structural}}.

A server rejects a call with an `invalidArguments` error ({{RFC8620}} Section 3.6.2) if `where` is not well-formed (other than by using an unknown operator), if exactly one of `language` and `expression` is present, or if `expression` is not a valid query in its language. It rejects a call with an `unsupportedFilter` error ({{RFC8620}} Section 5.5) if: `where` uses a `$`-prefixed name that is not an operator defined in this document (so that a future extension is reported as unsupported rather than malformed); `language` is not in the account's `languages` list; the expression uses `$text` and `supportsText` is `false`; or an operator object's `$collation` is not in `collationAlgorithms`, or the collation in effect does not support an operation the object applies to a string (equality, ordering, or prefix and substring matching).

# The Built-In Filter Language {#where}

Every server that advertises `urn:ietf:params:jmap:graph:search` supports the language in this section. Its syntax follows the JSON query filters of document databases such as {{DOCUMENTDB}}, with changes that make its results the same on every server.

## Expressions

An Expression is a JSON object. The expression is true if every one of its members is true. An empty object is always true. Each member is one of the following:

- A **field test**, whose name is a JSON Pointer ({{field-tests}}).
- A **logical operator**: `$and`, `$or`, or `$not` ({{logical}}).
- A **structural operator**: `$edge`, `$reachable`, `$from`, or `$to` ({{structural}}).
- The **text operator** `$text` ({{text}}).

A member name that begins with `$` and is not one of these is rejected as described in {{filter}}.

## Field Tests {#field-tests}

A field test's name is a JSON Pointer ({{RFC6901}}) that is resolved against the candidate's JSON representation, as returned by `/get`. For example, `/annotations/com.example.status` addresses the annotation `com.example.status`, and `/type` addresses a GraphEdge's type. JSON Pointer is used, rather than the dotted paths of document databases, because annotation keys commonly contain dots ({{JMAP-GRAPH}}). A pointer that does not resolve to a value leaves the field **missing**.

A field test's value is either an operator object or a plain value:

- An **operator object** is a JSON object at least one of whose member names begins with `$`. An object that mixes `$`-prefixed and other member names is not well-formed ({{filter}}). The test is true if every operator in it is true.
- Any other value is shorthand for `{"$eq": value}`.

## Comparison Operators {#comparison}

The following operators may appear in an operator object. "The field" means the value the pointer resolves to.

`$eq` (any value):
: The field is present and equal to the operand.

`$ne` (any value):
: Not `$eq`. True when the field is missing.

`$gt`, `$gte`, `$lt`, `$lte` (Number or String):
: The field and the operand are both numbers and compare numerically as stated, or are both strings and compare as stated under the test's collation ({{collation}}). Otherwise false.

`$in` (Array):
: The field is `$eq` to at least one element of the operand.

`$nin` (Array):
: Not `$in`. True when the field is missing.

`$exists` (Boolean):
: If `true`, the field is present (its value may be `null`); if `false`, it is missing.

`$prefix` (String):
: The field is a string that begins with the operand under the test's collation.

`$contains` (String):
: The field is a string that contains the operand as a substring under the test's collation.

`$collation` (String):
: Not a test. Names the collation algorithm used by this operator object's string operators. Its value is one of the `collationAlgorithms` in the core capability ({{RFC8620}} Section 2). When it is absent, strings are compared as described in {{collation}}.

Two JSON values are equal when they have the same JSON type and: both are `null`; both are the same Boolean; both are numbers with the same numeric value; both are strings that are equal under the test's collation; both are arrays of the same length whose elements are pairwise equal; or both are objects with the same member names whose values are pairwise equal. Values of different types are never equal and never ordered. In particular, `{"$eq": null}` is true only for an explicit `null`; a missing field is tested with `$exists`.

## Arrays {#arrays}

When the field is an array and the operand of `$eq`, `$gt`, `$gte`, `$lt`, `$lte`, `$prefix`, or `$contains` is not an array, the operator is true if it is true for at least one element of the field. `$in` is true if `$eq`, with this rule applied, is true for at least one element of the operand. `$ne` and `$nin` are the negations of `$eq` and `$in` with this rule applied, so `{"$ne": "x"}` on an array field is true only if no element equals `"x"`. Elements are not searched recursively.

## Logical Operators {#logical}

`$and` (Expression[]):
: True if every expression in the array is true. An empty array is true.

`$or` (Expression[]):
: True if at least one expression in the array is true. An empty array is false.

`$not` (Expression):
: True if the expression is false.

## Collation {#collation}

When `$collation` is absent, strings are compared as sequences of Unicode code points: two strings are equal only if they are identical, they are ordered by code point, and `$prefix` and `$contains` match code-point sequences exactly. This keeps results identical across servers and is correct for ids, which are case-sensitive. When `$collation` is present, every string operator in the same operator object, including `$eq`, `$ne`, `$in`, and `$nin`, uses it. A plain value (the `$eq` shorthand) always uses code-point comparison.

## Structural Operators {#structural}

Structural operators test a candidate's place in the graph. `$edge` and `$reachable` apply when the candidate is a GraphNode; `$from` and `$to` apply when it is a GraphEdge. An expression that uses one on the other kind of candidate is not well-formed ({{filter}}).

Only nodes and edges in the Graph named by the FilterCondition's `graphId`, and visible to the user ({{visibility}}), are considered.

`$edge` (EdgeTest):
: True if the candidate has at least one edge satisfying the EdgeTest. An EdgeTest is an object with the following members:

  - `direction` (String, required): `"out"` for edges whose `fromId` is the candidate, `"in"` for edges whose `toId` is the candidate, or `"any"` for both.
  - `type` (String, optional): the edge's `type` equals this value exactly.
  - `edge` (Expression, optional): an expression that the edge itself satisfies.
  - `node` (Expression, optional): an expression that the node at the other end of the edge satisfies.

`$reachable` (ReachTest):
: True if some node reachable from the candidate satisfies the test. A ReachTest is an object with the following members:

  - `direction` (String, required): as for EdgeTest; each step follows an edge in this direction.
  - `type` (String, optional): only edges whose `type` equals this value are followed.
  - `maxDepth` (UnsignedInt, required): the largest number of steps followed; at least 1.
  - `node` (Expression, optional): an expression that the reached node satisfies.

  A node is reachable if a path of 1 to `maxDepth` steps leads to it from the candidate. Cycles are permitted in the graph and do not affect the result: the test asks only whether such a path exists. The candidate itself is reachable only if a path of at least one step returns to it.

`$from` (Expression):
: The node named by the candidate edge's `fromId` satisfies the expression.

`$to` (Expression):
: The node named by the candidate edge's `toId` satisfies the expression.

Expressions inside structural operators may themselves contain structural operators, which is how a single expression describes a multi-step pattern.

## Traversal Depth {#depth}

The traversal depth of an expression is the largest, over every chain of nested structural operators, of the sum of the steps along the chain, where `$edge`, `$from`, and `$to` count as one step and `$reachable` counts as its `maxDepth`. A server MUST reject an expression whose traversal depth exceeds the advertised `maxDepth` with a `queryTooComplex` error ({{errors}}), so that clients can rely on the advertised limit.

## Full-Text Matching {#text}

`$text` (String):
: True if the candidate matches the given text. What constitutes a match (tokenization, stemming, language handling, which parts of node content are indexed) is server-defined, as it is for the `text` filter of `Email/query` in {{RFC8621}}. The server may consider the candidate's string-valued annotations and, for a GraphNode, its content.

A server advertises `supportsText: true` only if it supports this operator.

# Advertised Query Languages {#languages}

A server MAY support query languages registered in the "JMAP Graph Query Languages" registry ({{iana-languages}}) and lists them in the `languages` capability field. Each registration states how a Graph is presented to the language and what form a query takes. The following rules apply to all of them:

- The query sees only the Graph named by `graphId`, and only nodes and edges visible to the user ({{visibility}}).
- The query's result must consist of objects of the queried type. A query whose result consists of anything else is rejected with `invalidArguments`.
- The query must not change data. Servers SHOULD reject a query that would create, modify, or delete anything, with a `forbidden` error ({{RFC8620}} Section 3.6.2), because a change made by a query would bypass the rights checks and state tracking of `/set`.
- Servers SHOULD reject language features that reach outside the Graph, such as fetching remote resources or loading files ({{security}}).

## Property Graph Presentation {#property-graph}

The languages registered with this presentation (`gql`, `opencypher`, and `gremlin`) see the Graph as a property graph:

- Each GraphNode is a vertex (node) whose identifier is the GraphNode `id`. It has no label in {{GQL}} and {{OPENCYPHER}}, and the default vertex label `vertex` in {{GREMLIN}}, where every element has a label.
- Each GraphEdge is a directed edge from the vertex of `fromId` to the vertex of `toId`, whose identifier is the GraphEdge `id` and whose label is the GraphEdge `type`. An edge whose `type` is the empty string has no label in {{GQL}}, the default edge label `edge` in {{GREMLIN}}, and a server-defined relationship type in {{OPENCYPHER}}, where every relationship has exactly one type.
- Each annotation of a node or edge is a property whose name is the annotation key and whose value is the annotation value. How a JSON object or array value is represented is server-defined.
- Other GraphNode and GraphEdge properties are not presented.

A GraphNode/query expression returns vertices, and a GraphEdge/query expression returns edges: for {{GQL}} and {{OPENCYPHER}}, a query whose single returned column holds nodes or edges respectively; for {{GREMLIN}}, a traversal whose results are vertices or edges respectively. An expression that a language requires to be bound to a particular graph or traversal source is evaluated against the presented Graph.

## Document Presentation {#document}

The language registered with this presentation (`documentdb`) sees each GraphNode, during GraphNode/query, or each GraphEdge, during GraphEdge/query, as one document: its JSON representation as returned by `/get`, with the `id` value also present as `_id`. The expression is a query filter document, as used by find operations in {{DOCUMENTDB}}, serialized as a JSON string; aggregation pipelines are not accepted. The result is the set of documents the filter matches.

Because the dotted field paths of this language cannot address annotation keys that contain dots, how such keys are reached is server-defined. The built-in language ({{where}}) does not have this limitation.

## Server-Defined Presentation

For a language registered without one of the presentations above (`sparql`), how the Graph is presented to the language is server-defined in this revision. A future document may define it.

# Visibility {#visibility}

The nodes and edges a search considers are exactly those the user could fetch with `/get`. For an ordinary Graph, that is all of its contents. For a projected Graph ({{JMAP-GRAPH-PROJECTION}}), it is the contents visible to the user under that document. Servers SHOULD ensure that a node or edge the user cannot see has no effect on any search result, including through structural operators: otherwise a query such as "nodes with an edge to something" would reveal objects the user cannot read.

Field tests apply to properties added by other documents as well; for example, `/objectUri` addresses the source object of a node in a projected Graph.

# Query Changes {#query-changes}

A search result can change when objects other than the result objects change: a structural operator depends on the candidate's neighbors, and a foreign-language query may depend on anything in the Graph. Servers MAY return `cannotCalculateChanges` ({{RFC8620}} Section 5.6) for any `/queryChanges` call whose filter uses a property defined in this document.

# Errors {#errors}

This document defines one method-level error:

`queryTooComplex`:
: The query is valid, but the server will not evaluate it because it exceeds a server limit, such as `maxDepth` or a limit on evaluation time or work. The server MAY include a `description` property explaining the limit.

# Examples

## Annotation tests

Open or blocked nodes whose title starts with "Draft", ignoring case:

~~~json
["GraphNode/query", {
  "accountId": "A1",
  "filter": {
    "graphId": "G1",
    "where": {
      "/annotations/com.example.status": {
        "$in": ["open", "blocked"]
      },
      "/annotations/com.example.title": {
        "$prefix": "draft",
        "$collation": "i;unicode-casemap"
      }
    }
  }
}, "0"]
~~~

## Descendants

All nodes below `N1` in a tree built from `com.example.child` edges (from parent to child), up to ten levels down. A node is below `N1` if `N1` is reachable from it by following `com.example.child` edges backwards:

~~~json
["GraphNode/query", {
  "accountId": "A1",
  "filter": {
    "graphId": "G1",
    "where": {
      "$reachable": {
        "direction": "in",
        "type": "com.example.child",
        "maxDepth": 10,
        "node": { "/id": "N1" }
      }
    }
  }
}, "0"]
~~~

## Edges between matching nodes

`com.example.depends-on` edges from an open node to a closed one:

~~~json
["GraphEdge/query", {
  "accountId": "A1",
  "filter": {
    "graphId": "G1",
    "type": "com.example.depends-on",
    "where": {
      "$from": { "/annotations/com.example.status": "open" },
      "$to": { "/annotations/com.example.status": "closed" }
    }
  }
}, "0"]
~~~

## An advertised language

The previous example's source nodes, found with openCypher on a server that lists `opencypher` in `languages`. The query is:

~~~
MATCH (a)-[:`com.example.depends-on`]->(b)
WHERE a.`com.example.status` = 'open'
  AND b.`com.example.status` = 'closed'
RETURN a
~~~

It is carried as the `expression` string, here abbreviated as `"MATCH ... RETURN a"` (the string holds the whole query, on one line or with `\n` separators):

~~~json
["GraphNode/query", {
  "accountId": "A1",
  "filter": {
    "graphId": "G1",
    "language": "opencypher",
    "expression": "MATCH ... RETURN a"
  }
}, "0"]
~~~

# Security Considerations {#security}

## Resource Consumption

Searches can be expensive: `$reachable` explores many paths, `$contains` and `$text` may require scanning content, and advertised languages can express queries of unbounded cost. The advertised `maxDepth` bounds built-in traversal. Servers SHOULD also bound evaluation time and work for every search, rejecting or abandoning queries that exceed their limits with `queryTooComplex`, and MAY apply the rate limits they apply to other methods.

The built-in language has no regular-expression operator, because regular-expression dialects differ between implementations and some patterns take time exponential in the input.

## Foreign Language Features

Some registered languages include features that act outside the data being queried: SPARQL's `SERVICE` keyword sends a query to a remote endpoint ({{SPARQL11-FED}}), openCypher implementations commonly offer `LOAD CSV FROM` with a URL, and Gremlin implementations may accept lambdas or scripts that run arbitrary code. These features can be used for server-side request forgery or code execution. Servers SHOULD accept only the subset of each language needed to query the presented Graph, and SHOULD reject queries using any feature that fetches remote resources, reads files, or runs user-supplied code.

## Writes Through Queries

Several registered languages can modify data (for example, GQL `INSERT`, openCypher `CREATE` and `DELETE`, and Gremlin `addV()` and `drop()`). {{languages}} requires that a search not change data, and servers SHOULD enforce this by rejecting such queries before evaluation, not by evaluating them in a way that discards their changes.

## Information Disclosure

Search reveals information through which objects match, not only through what the results contain. The visibility requirement of {{visibility}} keeps hidden objects from influencing results. Full-text indexing of node content stores that content in a second form; deployments should treat indexes with the same care as the content.

# IANA Considerations {#iana}

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP Capabilities" registry:

Capability Name:
: `urn:ietf:params:jmap:graph:search`

Intended Use:
: common

Change Controller:
: IETF

Specification document:
: This document.

Security and privacy considerations:
: See {{security}} of this document.

## JMAP Error Code Registration

IANA is requested to register the following entry in the "JMAP Error Codes" registry ({{RFC8620}} Section 9.5):

JMAP Error Code:
: `queryTooComplex`

Intended Use:
: common

Change Controller:
: IETF

Reference:
: This document, {{errors}}

Description:
: The query is valid, but the server will not evaluate it because it exceeds a server limit.

## JMAP Graph Query Languages Registry {#iana-languages}

IANA is requested to create the "JMAP Graph Query Languages" registry in the "JSON Meta Application Protocol (JMAP)" registry group. The registration policy is Expert Review ({{RFC8126, Section 4.5}}). The designated expert checks that the registration identifies the language precisely enough for independent implementations (including its version or edition), states how a Graph is presented to it and what a query returns, and addresses the language's features that act outside the presented Graph.

Each registration has the following fields:

Identifier:
: A short string of lowercase ASCII letters, digits, and hyphens.

Language:
: The language, with its version or edition.

Presentation:
: How a Graph is presented to the language and what form an expression takes: a reference to {{property-graph}} or {{document}} of this document, a definition in the registering document, or "server-defined".

Reference:
: The document that defines the language, and the document that registers it.

Initial registrations:

| Identifier | Language | Presentation | Reference |
|---|---|---|---|
| gql | ISO/IEC 39075:2024 GQL {{GQL}} | {{property-graph}} | This document |
| opencypher | openCypher 9 {{OPENCYPHER}} | {{property-graph}} | This document |
| gremlin | Gremlin, Apache TinkerPop 3.7 {{GREMLIN}} | {{property-graph}} | This document |
| documentdb | DocumentDB 1.0 (Linux Foundation) query filter documents {{DOCUMENTDB}} | {{document}} | This document |
| sparql | SPARQL 1.1 Query Language {{SPARQL11}} | Server-defined | This document |

--- back

# Design Notes

## Influences

- **Document-database query filters**, as in MongoDB and the Linux Foundation's {{DOCUMENTDB}}, supplied the syntax of the built-in language: a JSON object whose members are field tests with `$`-prefixed operators. JSON filters fit JMAP, whose own filters are JSON objects.
- **MongoDB's `$graphLookup`** ({{MONGODB-GRAPHLOOKUP}}) suggested a bounded recursive traversal operator; `$reachable` is its filter-shaped equivalent.
- **Property graph languages** ({{GQL}}, {{OPENCYPHER}}, {{GREMLIN}}) share a data model with {{JMAP-GRAPH}}, which makes a single presentation for all three possible.

## Differences from Document-Database Filters

The built-in language departs from document-database filters wherever their behavior varies between implementations or would need a long specification:

- Fields are addressed by JSON Pointer, not dotted paths, because annotation keys contain dots.
- `{"$eq": null}` does not match a missing field.
- There is no `$regex`, `$where`, `$expr`, or `$elemMatch`.

## Why No Language Is Mandatory

Of the established property-graph query languages, the only one standardized by a standards body ({{GQL}}) is large and not freely available, and the others are defined by their implementations or by a single vendor. {{SPARQL11}} is a W3C Recommendation, but it is designed for RDF rather than property graphs. Requiring any of them would make the built-in baseline expensive. Advertising them lets servers that already have one offer it.

# Acknowledgements

The author thanks the designers of the query languages listed in this document, whose work it presents over JMAP rather than replaces.
