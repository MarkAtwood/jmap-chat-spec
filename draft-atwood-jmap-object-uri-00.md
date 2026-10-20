---
title: JMAP Object URIs
abbrev: JMAP Object URIs
docname: draft-atwood-jmap-object-uri-00
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
  RFC3986:
  RFC5234:
  RFC8620:
  RFC9404:

informative:
  RFC3553:
  RFC7595:
  RFC8126:
  JMAP-CHAT:
    title: "JMAP for Chat"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-00
    date: 2026
  JMAP-CHAT-FED:
    title: "JMAP Chat Federation"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-federation-00
    date: 2026

--- abstract

This document defines the `jmap` URI scheme and, with it, JMAP object URIs: references of the form `jmap:<type>:<accountId>:<objectId>` that name one JMAP ({{RFC8620}}) object by data type, account, and id, for use in fields whose value is a single URI. It also creates the "JMAP Object URI Types" registry, which maps the type names used in these URIs to JMAP data types. A JMAP object URI is interpreted relative to the JMAP session or federation peer from which it was received; it is not globally unique and cannot be dereferenced on its own.

--- middle

# Introduction

{{RFC8620}} refers to an object by two separate values: an account id and an object id, with the data type implied by the method being called. Several JMAP extensions need to carry such a reference in a field whose value is a single URI. For example, a JMAP Chat ({{JMAP-CHAT}}) MessageAction or Endpoint has a `uri` field that may point at a calendar event, a principal, or a task. This document defines a URI scheme for such references.

## Why a New Scheme

Existing schemes do not fit. An `https` URL names a network resource rather than an object within a JMAP account. A URN in the `urn:ietf:params:jmap` sub-namespace ({{RFC8620, Section 9.3}}) cannot carry account and object ids, because names in that namespace must be persistent and unique ({{RFC3553}}), and JMAP ids are neither. A new scheme, defined as a context-relative reference, avoids both problems.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Terminology from {{RFC8620}} is used throughout.

# JMAP Object URIs {#object-uri}

A JMAP object URI refers to one JMAP object by type, account, and id. It uses the `jmap` URI scheme ({{RFC3986}}), which this document defines and registers ({{iana-uri-scheme}}). A JMAP object URI matches the following syntax ({{RFC5234}}):

~~~abnf
jmap-object-uri = "jmap:" type-name ":" account-id ":" object-id
type-name       = 1*( %x61-7A / DIGIT )
                  ; registered in "JMAP Object URI Types"
account-id      = 1*255id-char
object-id       = 1*255id-char
id-char         = ALPHA / DIGIT / "-" / "_"
                  ; the Id character set of RFC 8620, Section 1.2
~~~

`type-name` is a type name registered in the "JMAP Object URI Types" registry ({{iana-uri-types}}), which maps it to the JMAP data type it stands for. `account-id` is the JMAP accountId of the account holding the object, and `object-id` is the object's `id`. For example, `jmap:calendarevent:A13824:E6B2` refers to the CalendarEvent with id `E6B2` in account `A13824`.

## Interpretation

A JMAP object URI is a reference interpreted relative to the JMAP session or federation peer from which it was received. It is not globally unique: {{RFC8620}} does not make account ids globally unique, so equal URIs received from two different servers can refer to different objects.

A JMAP object URI is therefore meaningful only on the server whose account it names. A server that passes data received from another server, such as a federation peer, on to its own clients or to a third server MUST NOT pass a JMAP object URI in that data through unchanged, because its account id would then be resolved in the wrong context, either failing or naming a different object. The protocol that carries the URI specifies what the server does instead, such as removing the field or the structure that holds it.

A JMAP object URI cannot be dereferenced on its own. A client resolves it, in the context from which it was received, by looking up the type name's Data Type in the "JMAP Object URI Types" registry and calling that data type's `/get` method on the named account, including in `using` the capability that the "JMAP Data Types" registry ({{RFC9404, Section 6.3}}) lists for that data type.

## Equivalence

Two JMAP object URIs are equal if they are identical, except that the scheme name is case-insensitive ({{RFC3986, Section 3.1}}). The other components are compared case-sensitively, as JMAP ids require. No component is ever percent-encoded, because every permitted character is unreserved ({{RFC3986, Section 2.3}}).

## Other Forms

Other forms of `jmap` URI may be defined by documents that update the scheme registration ({{iana-uri-scheme}}). `jmap` URIs whose scheme-specific part begins with `//` (an authority component, {{RFC3986, Section 3.2}}) are reserved for such forms; they never match `jmap-object-uri`, because a type name cannot contain `/`. A receiver treats a `jmap` URI that matches no form it implements as unresolvable.

## Use

JMAP object URIs are for object references within JMAP method calls, including calls between servers such as the federation methods of {{JMAP-CHAT-FED}}. Servers SHOULD NOT expose these URIs to non-JMAP systems without transformation.

A document that uses JMAP object URIs for a data type registers that data type's type name in the "JMAP Object URI Types" registry.

# Security Considerations {#security}

A `jmap` URI names no network location, so resolving one causes no network access beyond the JMAP session in which it is resolved; there is no fetch from a host named in the URI. A JMAP object URI received from another party is untrusted input: its account id and object id are resolved only within the receiving session's access controls, and a reference to an object the user cannot read yields the same `notFound` result as a reference to one that does not exist. Because equal URIs from different servers can refer to different objects, servers do not relay them unchanged between contexts ({{object-uri}}); without that rule, a client could resolve a relayed URI against its own server and be shown an unrelated object.

A JMAP object URI discloses an account id and an object id to every party that receives it: for example, every member of a JMAP Chat Space in whose messages it appears and, through federation, every peer server. {{RFC8620}} does not require account ids to be opaque, so a deployment that derives them from user names, tenant names, or other personal data discloses that data through every URI it emits; such servers SHOULD use opaque ids.

Of the considerations in {{RFC3986, Section 7}}, Section 7.1 applies as described in {{object-uri}}, and Section 7.5 as just described. Sections 7.2, 7.4, and 7.6 do not apply, because the scheme has no authority, host, port, or userinfo component, and Section 7.3 does not apply, because no component needs percent-encoding.

# IANA Considerations {#iana}

## URI Scheme Registration {#iana-uri-scheme}

IANA is requested to register the following entry in the "Uniform Resource Identifier (URI) Schemes" registry, following the template of {{RFC7595, Section 7.4}}:

Scheme name:
: jmap

Status:
: Permanent

Applications/protocols that use this scheme name:
: Extensions of JMAP ({{RFC8620}}) that refer to individual JMAP objects within JMAP method calls, such as JMAP Chat MessageActions.

Contact:
: Mark Atwood <mark@reviewcommit.com>

Change controller:
: IETF

References:
: This document. Security and privacy considerations: {{security}}.

## JMAP Object URI Types Registry {#iana-uri-types}

IANA is requested to create the "JMAP Object URI Types" registry in the "JSON Meta Application Protocol (JMAP)" registry group. It lists the type names used in JMAP object URIs ({{object-uri}}). The registration policy is Specification Required ({{RFC8126, Section 4.6}}).

The designated expert checks that the Data Type is registered in the "JMAP Data Types" registry ({{RFC9404, Section 6.3}}), that the type name matches the `type-name` rule in {{object-uri}} and is the lowercased name of that data type, and that no existing entry has the same type name.

Each registration has the following fields:

Type Name:
: The type name, matching the `type-name` rule.

Data Type:
: The name of the data type, exactly as registered in the "JMAP Data Types" registry.

Reference:
: The document that defines the type's use in JMAP object URIs.

The registry has no initial entries. Documents that use JMAP object URIs register the type names they need.

--- back

# Acknowledgements

The author thanks the authors of {{RFC8620}}, on whose object model these references are built.
