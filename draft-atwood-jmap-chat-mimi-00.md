---
title: JMAP Chat MIMI Mapping
abbrev: JMAP Chat MIMI
docname: draft-atwood-jmap-chat-mimi-00
category: std
stream: independent

ipr: trust200902
date: 2026

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
  -
    fullname: Mark Atwood
    email: mark@reviewcommit.com

normative:
  RFC2119:
  RFC4648:
  RFC8174:
  JMAP-CHAT:
    title: JMAP for Chat
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-00
    date: 2026
  MIMI-CONTENT:
    title: "More Instant Messaging Interoperability (MIMI) message content"
    author:
      fullname: R. Mahy
    seriesinfo:
      Internet-Draft: draft-ietf-mimi-content-09
    date: 2026
  ULID:
    title: Universally Unique Lexicographically Sortable Identifier
    target: https://github.com/ulid/spec

informative:
  RFC9420:
  MIMI-PROTOCOL:
    title: "More Instant Messaging Interoperability (MIMI) using HTTPS and MLS"
    author:
      - fullname: R. L. Barnes
      - fullname: M. Hodgson
      - fullname: K. Kohbrok
      - fullname: R. Mahy
      - fullname: T. Ralston
      - fullname: R. Robert
    seriesinfo:
      Internet-Draft: draft-ietf-mimi-protocol-06
    date: 2026
  MIMI-MESSAGE-STATUS:
    title: "A Message Status format for the More Instant Messaging Interoperability (MIMI) content format"
    author:
      fullname: R. Mahy
    seriesinfo:
      Internet-Draft: draft-mahy-mimi-message-status-01
    date: 2026
  JMAP-CHAT-FED:
    title: JMAP Chat Federation
    target: https://datatracker.ietf.org/doc/draft-atwood-jmap-chat-federation/

--- abstract

This document defines how a gateway converts messages between the IETF More Instant Messaging Interoperability (MIMI) content format ({{MIMI-CONTENT}}) and JMAP Chat ({{JMAP-CHAT}}). It specifies how MIMI message identifiers, edits, topics, media reactions, and message statuses map onto JMAP Chat Messages, so that independent gateways derive the same JMAP Chat identifiers for the same MIMI messages without coordination.

--- middle

# Introduction

{{JMAP-CHAT}} and MIMI ({{MIMI-PROTOCOL}}, {{MIMI-CONTENT}}) address different layers of messaging interoperability: MIMI federates large providers through hub-and-spoke rooms, while JMAP Chat defines a client-server API and, in {{JMAP-CHAT-FED}}, mailbox-to-mailbox federation. A deployment that participates in both needs a gateway that converts messages between them.

This document collects the conversion rules that such a gateway follows. Servers and clients that never convert MIMI messages are not affected by it. Keeping these rules separate from {{JMAP-CHAT}} lets them track revisions of the MIMI drafts without revising the core specification.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Terminology from {{JMAP-CHAT}} and {{MIMI-CONTENT}} is used throughout. A "converting party" is any server or client that converts `application/mimi-content` messages to JMAP Chat Messages.

# Message Identifiers {#message-ids}

A converting party MUST derive each message's `senderMsgId` from its MIMI message ID: the 16 octets that follow the MIMI message ID's first (hash algorithm) octet, read as a 128-bit big-endian integer and encoded as a ULID {{ULID}}. Every party that holds a MIMI message ID therefore derives the same `senderMsgId` without coordination, including for a referenced message it never received. The timestamp component of such a ULID carries no meaning; ordering uses the receiving mailbox's Message `id` ({{JMAP-CHAT}}).

# Edits {#edits}

A MIMI edit, whose `replaces` field always names the first version of the message {{MIMI-CONTENT}}, maps to an edit of the message whose `senderMsgId` is derived from `replaces`. The converting party records the `senderMsgId` derived from each edit's own MIMI message ID as an alias of that message, so a MIMI reply or reaction that names a later version refers to the message as a whole.

# Topics {#topics}

MIMI's `topicId` {{MIMI-CONTENT}}, a byte string, maps to the Message `topicId` field ({{JMAP-CHAT}}) as its unpadded base64url encoding ({{RFC4648, Section 5}}).

# Reactions {#reactions}

A MIMI media reaction (a reaction that is not an emoji) is carried in JMAP Chat as a reply containing the media, as {{MIMI-CONTENT}} permits; JMAP Chat Reactions ({{JMAP-CHAT}}) remain emoji and custom emoji.

# Message Status {#status}

MIMI message statuses `expired`, `hidden`, and `error` {{MIMI-MESSAGE-STATUS}} are not read acknowledgements, so a converting party does not set `readAt` or a ReadDisposition for them.

# Security Considerations {#security}

A converting party that relays end-to-end encrypted MIMI messages ({{RFC9420}}) into JMAP Chat must decrypt them to convert their content, and so holds plaintext. It SHOULD join each MLS group as a member visible to the other members, so that users can see that a gateway holds plaintext, rather than obtaining keys by any path that does not appear in the group membership.

Derived `senderMsgId` values are only as unique as the MIMI message IDs they come from. A converting party MUST NOT accept a MIMI message whose derived `senderMsgId` equals that of a different message already stored in the same Chat; it rejects the message rather than overwriting or merging the stored one.

Alias records ({{edits}}) let a MIMI reply, reaction, or edit that names a later version reach the original message. A converting party MUST verify that the sender of an edit is the sender of the original message before recording its alias, so that a third party cannot attach an alias to, and thereby edit or retract, someone else's message.

# IANA Considerations {#iana}

This document has no IANA actions.

--- back

# Acknowledgements

The author thanks the MIMI working group, and the authors of {{MIMI-CONTENT}} and {{MIMI-PROTOCOL}} in particular, for the content format and protocol this mapping targets.
