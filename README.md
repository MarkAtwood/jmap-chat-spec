# JMAP Chat Specification

A suite of IETF-style Internet-Draft specifications for JMAP Chat — a real-time chat protocol built as a JMAP capability extension on top of [RFC 8620](https://www.rfc-editor.org/rfc/rfc8620).

**Author:** Mark Atwood \<mark@reviewcommit.com\>

---

## What Is JMAP Chat?

JMAP Chat defines a complete messaging protocol in the same vein as [RFC 8621](https://www.rfc-editor.org/rfc/rfc8621) (JMAP for Mail). Where JMAP Mail gives you email over JMAP, JMAP Chat gives you real-time messaging: direct messages, group chats, threaded channels, presence, typing indicators, reactions, read receipts, and push notifications — all expressed as JMAP data types and methods.

The protocol is designed for two deployment topologies:

**Mailbox-per-user (federated):** Each participant runs their own JMAP server storing only their own messages. Servers communicate directly using the server-to-server Peer/* methods defined in the federation spec. No central operator; no central message store.

**Relay:** A shared server routes messages between clients. The relay handles only opaque ciphertext in end-to-end encrypted deployments — it never sees plaintext content.

Both topologies use the same client-facing API.

---

## Specifications

### Core

| Draft | Capability | What It Covers |
|---|---|---|
| [draft-atwood-jmap-chat-00](draft-atwood-jmap-chat-00.md) | `urn:ietf:params:jmap:chat` | Core data types, methods, push events, blob storage |
| [draft-atwood-jmap-chat-federation-00](draft-atwood-jmap-chat-federation-00.md) | — | Server-to-server Peer/* methods, peer discovery, federation auth |

### Extensions

| Draft | Capability | What It Covers |
|---|---|---|
| [draft-atwood-jmap-chat-push-00](draft-atwood-jmap-chat-push-00.md) | `urn:ietf:params:jmap:chat:push` | Inline push payloads (`ChatMessagePush`) for mobile/background clients |
| [draft-atwood-jmap-chat-wss-00](draft-atwood-jmap-chat-wss-00.md) | `urn:ietf:params:jmap:chat:websocket` | WebSocket transport: ephemeral typing and presence events |
| [draft-atwood-jmap-chat-filenode-00](draft-atwood-jmap-chat-filenode-00.md) | `urn:ietf:params:jmap:chat:filenode` | Space-scoped shared file storage via JMAP FileNode |
| [draft-atwood-jmap-chat-calendars-00](draft-atwood-jmap-chat-calendars-00.md) | `urn:ietf:params:jmap:chat:calendars` | Binds Spaces to JMAP Calendars; surfaces CalendarEvents, RSVP, and availability in chat |
| [draft-atwood-jmap-chat-tasks-00](draft-atwood-jmap-chat-tasks-00.md) | `urn:ietf:params:jmap:chat:tasks` | Binds Spaces to JMAP Tasks TaskLists; surfaces Tasks in chat with Task↔Chat back-references |
| [draft-atwood-jmap-chat-did-00](draft-atwood-jmap-chat-did-00.md) | `urn:ietf:params:jmap:chat:did` | Decentralized Identifier support: mandatory DID methods, ChatContact extensions, federation auth |
| [draft-atwood-jmap-cid-00](draft-atwood-jmap-cid-00.md) | `urn:ietf:params:jmap:cid` | SHA-256 content identifiers on blob upload responses and FileNode objects |

---

## Feature Summary

**Conversation types**
- Direct chats (one-to-one)
- Group chats (multi-party, with admin/member roles)
- Spaces — named workspaces with channels, role-based permissions, and category organization (analogous to Discord servers or Slack workspaces)

**Messages**
- Plain text, Markdown, and a structured rich-text body format (`application/jmap-chat-rich`)
- File attachments with SHA-256 integrity verification
- @mentions with byte-offset annotations
- Message editing with full revision history
- Soft deletion (tombstone) and hard deletion (sender-controlled expiry, burn-on-read)
- Reactions (emoji, including Space-scoped custom emoji)
- Threads (optional, server-advertised)
- Out-of-band action invitations (video calls, payments, file links)

**Real-time**
- Typing indicators (ephemeral, rate-limited, opt-out per chat)
- Presence: online/away/busy/invisible/offline with custom status text and emoji
- Read receipts with privacy opt-out (per-account and per-chat)
- WebSocket transport for low-latency bidirectional communication

**Push**
- Inline `ChatMessagePush` payloads — mobile clients display notifications without a follow-up request
- Mention-aware urgency (different priority for @mentions vs. regular messages)
- E2EE-aware: no body snippets for encrypted message bodies
- Fallback to standard `StateChange` under rate limiting

**Privacy and security**
- End-to-end encrypted relay deployments (server routes opaque ciphertext only)
- Receipt sharing opt-out (server suppresses outbound `Peer/receipt` calls)
- Typing indicator suppression per chat (`receiveTypingIndicators`)
- Contact blocking (messages silently dropped server-side)
- SSRF-hardened blob fetch in federation

**Spaces**
- Role hierarchy with named permissions (view, send, pin, manage_channels, ban, etc.)
- Per-channel permission overrides
- Invite codes with expiry and use limits
- Bans with optional expiry
- Public/publicly-previewable visibility modes
- Optional shared file tree via JMAP FileNode

---

## Implementer Guides

Non-normative companion documents for implementers. These explain *how* to implement the specs, not just *what* they require.

| Guide | For |
|---|---|
| [jmap-chat-implementer-guide.md](jmap-chat-implementer-guide.md) | Server and client implementers: governance, authorization, identity, and deployment-defined posture decisions for the core spec |
| [jmap-push-platform-guide.md](jmap-push-platform-guide.md) | Server implementers: delivering JMAP push to FCM, APNs, ADM, HPK, MiPush, WNS, and Web Push |
| [jmap-chat-push-platform-guide.md](jmap-chat-push-platform-guide.md) | Server implementers: encoding `ChatMessagePush` payloads for each platform (supplement to the above) |
| [jmap-chat-wss-guide.md](jmap-chat-wss-guide.md) | Client and server implementers: WebSocket connection lifecycle, event handling, fan-out architecture |
| [jmap-chat-federation-guide.md](jmap-chat-federation-guide.md) | Server operators: running federation as a service — peer auth, allowlists, abuse mitigation, observability |
| [jmap-chat-filenode-guide.md](jmap-chat-filenode-guide.md) | Server operators: running Space file storage at scale — backends, scanning, quotas, previews, grace-period |
| [jmap-chat-calendars-guide.md](jmap-chat-calendars-guide.md) | Server and client implementers: deployment posture for calendar binding, RSVP, availability, ICS parsing |
| [jmap-chat-tasks-guide.md](jmap-chat-tasks-guide.md) | Server and client implementers: deployment posture for TaskList binding, Task↔Chat back-references, workflow |
| [jmap-chat-did-guide.md](jmap-chat-did-guide.md) | Server and client implementers: federation auth realization, DID resolution caching, DID document conventions, provisioning, recovery |

---

## Relationship to Other Standards

**Built on:** [RFC 8620](https://www.rfc-editor.org/rfc/rfc8620) (JMAP Core). All JMAP Chat data types, methods, and push mechanisms are JMAP capabilities — clients use the standard JMAP request/response framing.

**Analogous to:** [RFC 8621](https://www.rfc-editor.org/rfc/rfc8621) (JMAP for Mail). JMAP Chat is to instant messaging what JMAP Mail is to email.

**Complementary to:** [MIMI](https://datatracker.ietf.org/wg/mimi/about/) (More Instant Messaging Interoperability). MIMI targets provider-to-provider federation between large existing platforms under regulatory mandates; JMAP Chat targets decentralized mailbox-per-user federation and JMAP-native deployments. The two protocols are not interoperable at the federation layer but are not in conflict.

**Optionally integrates with:**
- [JMAP FileNode](https://datatracker.ietf.org/doc/draft-ietf-jmap-filenode/) — hierarchical file storage for Space file trees
- [JMAP Blob Management](https://datatracker.ietf.org/doc/draft-ietf-jmap-blobext/) — extended blob operations
- [MLS / RFC 9420](https://www.rfc-editor.org/rfc/rfc9420) — end-to-end encryption key schedule for relay deployments
- [JMAP Quotas](https://datatracker.ietf.org/doc/draft-ietf-jmap-quotas/), [JMAP Metadata](https://datatracker.ietf.org/doc/draft-ietf-jmap-metadata/), [JMAP Calendars](https://datatracker.ietf.org/doc/draft-ietf-jmap-calendars/), [JMAP Tasks](https://datatracker.ietf.org/doc/draft-ietf-jmap-tasks/)

---

## Repository Layout

```
draft-atwood-jmap-chat-00.md            Core spec
draft-atwood-jmap-chat-federation-00.md Federation (server-to-server)
draft-atwood-jmap-chat-push-00.md       Push notification extension
draft-atwood-jmap-chat-wss-00.md        WebSocket extension
draft-atwood-jmap-chat-filenode-00.md   File storage extension
draft-atwood-jmap-chat-calendars-00.md  Calendar binding extension
draft-atwood-jmap-chat-tasks-00.md      Task binding extension
draft-atwood-jmap-chat-did-00.md        DID-based identity extension
draft-atwood-jmap-cid-00.md             Blob content identifiers
jmap-chat-implementer-guide.md          Core implementer's guide (non-normative)
jmap-push-platform-guide.md             Platform delivery guide (non-normative)
jmap-chat-push-platform-guide.md        Chat push supplement (non-normative)
jmap-chat-wss-guide.md                  WebSocket implementer's guide (non-normative)
jmap-chat-federation-guide.md           Federation implementer's guide (non-normative)
jmap-chat-filenode-guide.md             File storage implementer's guide (non-normative)
jmap-chat-calendars-guide.md            Calendars implementer's guide (non-normative)
jmap-chat-tasks-guide.md                Tasks implementer's guide (non-normative)
jmap-chat-did-guide.md                  DID implementer's guide (non-normative)
references/                             Referenced IETF drafts and RFCs
```

The drafts are the normative specifications. The guides are non-normative; when a guide conflicts with a draft, the draft is authoritative.
