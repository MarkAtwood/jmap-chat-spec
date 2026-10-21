# JMAP Chat and Companion Specifications

A suite of IETF-style Internet-Draft specifications built as JMAP capability extensions on top of [RFC 8620](https://www.rfc-editor.org/rfc/rfc8620). The centre of the suite is JMAP Chat, a real-time chat protocol. Around it sit companion capabilities for video/voice calling (VTC), shared 3D/2D spaces (Scene), general-purpose graphs (Graph), and small shared building blocks (content identifiers, Decentralized Identifiers, object URIs).

Each capability is optional and advertised in the JMAP Session object. A deployment implements the ones it needs.

**Author:** Mark Atwood \<mark@reviewcommit.com\>

---

## What Is JMAP Chat?

JMAP Chat defines a complete messaging protocol in the same vein as [RFC 8621](https://www.rfc-editor.org/rfc/rfc8621) (JMAP for Mail). Where JMAP Mail gives you email over JMAP, JMAP Chat gives you real-time messaging: direct messages, group chats, multi-channel Spaces, presence, typing indicators, reactions, read receipts, and push notifications, all expressed as JMAP data types and methods.

The protocol is designed for two deployment topologies:

**Mailbox-per-user (federated):** Each participant's home server stores only that participant's messages. Servers find each other through `/.well-known/jmap`, sign every request with Ed25519 HTTP Message Signatures ([RFC 9421](https://www.rfc-editor.org/rfc/rfc9421)), and exchange messages and events through the server-to-server `Peer/*` methods. There is no central operator and no central message store.

**Relay:** A shared server routes messages between clients. In end-to-end encrypted deployments the relay handles only opaque ciphertext.

Both topologies use the same client-facing API.

---

## Specifications

The drafts are the normative specifications. Every draft is at revision `-00`.

### JMAP Chat

| Draft | Capability | What It Covers |
|---|---|---|
| [draft-atwood-jmap-chat-00](draft-atwood-jmap-chat-00.md) | `urn:ietf:params:jmap:chat` | Core: 21 data types (Chat, Message, ChatContact, Space, SpaceRole, …), owner methods, push events, blob storage |
| [draft-atwood-jmap-chat-federation-00](draft-atwood-jmap-chat-federation-00.md) | — (extends the core capability) | Server-to-server: peer discovery, signed requests, `Peer/deliver`, `Peer/receipt`, `Peer/typing`, `Peer/retract`, `Peer/groupUpdate`, optional federated presence |
| [draft-atwood-jmap-chat-push-00](draft-atwood-jmap-chat-push-00.md) | `urn:ietf:params:jmap:chat:push` | Inline push payloads (`ChatMessagePush`) for mobile/background clients; mention-aware urgency |
| [draft-atwood-jmap-chat-wss-00](draft-atwood-jmap-chat-wss-00.md) | `urn:ietf:params:jmap:chat:websocket` | WebSocket ([RFC 8887](https://www.rfc-editor.org/rfc/rfc8887)) delivery of ephemeral typing and presence events |
| [draft-atwood-jmap-chat-filenode-00](draft-atwood-jmap-chat-filenode-00.md) | `urn:ietf:params:jmap:chat:filenode` | Space-scoped shared file tree via JMAP FileNode; Space roles mapped to file rights |
| [draft-atwood-jmap-chat-calendars-00](draft-atwood-jmap-chat-calendars-00.md) | `urn:ietf:params:jmap:chat:calendars` | Binds Spaces to JMAP Calendars; CalendarEvents, RSVP, and availability in chat |
| [draft-atwood-jmap-chat-tasks-00](draft-atwood-jmap-chat-tasks-00.md) | `urn:ietf:params:jmap:chat:tasks` | Binds Spaces to JMAP Tasks TaskLists; Task↔Chat back-references |
| [draft-atwood-jmap-chat-did-00](draft-atwood-jmap-chat-did-00.md) | `urn:ietf:params:jmap:chat:did` | DIDs as ChatContact identities and federation principals; binds the federation signing key to the DID. Requires `urn:ietf:params:jmap:did` |
| [draft-atwood-jmap-chat-mimi-00](draft-atwood-jmap-chat-mimi-00.md) | `urn:ietf:params:jmap:chat:mimi` | MIMI content ↔ JMAP Chat Message mapping for gateways, plus a capability for a JMAP Chat server to act as an MLS client's MIMI provider |

### JMAP VTC (video/voice teleconferencing)

| Draft | Capability | What It Covers |
|---|---|---|
| [draft-atwood-jmap-vtc-00](draft-atwood-jmap-vtc-00.md) | `urn:ietf:params:jmap:vtc` | Call state: VTCCall, VTCParticipant, VTCRecording, VTCLivestream; ring push. Media transport is deployment-chosen. Optional links to Chat and Calendars |
| [draft-atwood-jmap-vtc-wss-00](draft-atwood-jmap-vtc-wss-00.md) | `urn:ietf:params:jmap:vtc:websocket` | Call-scoped WebSocket subscription to ephemeral in-call events |

### JMAP Scene (spatial environments)

| Draft | Capability | What It Covers |
|---|---|---|
| [draft-atwood-jmap-scene-00](draft-atwood-jmap-scene-00.md) | `urn:ietf:params:jmap:scene` | Spatial state: SceneRegion, SceneObject, SceneAvatar, SceneAsset; spatial query filters; `SceneObject/interact`. Rendering and simulation are deployment-chosen. Optional links to Chat and VTC |
| [draft-atwood-jmap-scene-wss-00](draft-atwood-jmap-scene-wss-00.md) | `urn:ietf:params:jmap:scene:websocket` | Region-scoped WebSocket subscription to discrete spatial events (enter/exit, place/remove, interact) |

### JMAP Graph

| Draft | Capability | What It Covers |
|---|---|---|
| [draft-atwood-jmap-graph-00](draft-atwood-jmap-graph-00.md) | `urn:ietf:params:jmap:graph` | General-purpose shareable directed graphs (Graph, GraphNode, GraphEdge) with no built-in semantics |
| [draft-atwood-jmap-graph-projection-00](draft-atwood-jmap-graph-projection-00.md) | `urn:ietf:params:jmap:graph:projection` | Optional projection of an account's other JMAP data as a Graph |
| [draft-atwood-jmap-graph-search-00](draft-atwood-jmap-graph-search-00.md) | `urn:ietf:params:jmap:graph:search` | Built-in JSON filter language plus advertised query languages (GQL, openCypher, Gremlin, DocumentDB, SPARQL) |
| [draft-atwood-jmap-graph-edit-00](draft-atwood-jmap-graph-edit-00.md) | `urn:ietf:params:jmap:graph:edit` | Live co-editing of node content: server-ordered op log the server never merges, snapshots, WebSocket stream of ops and cursors |

### Shared building blocks

| Draft | Capability | What It Covers |
|---|---|---|
| [draft-atwood-jmap-cid-00](draft-atwood-jmap-cid-00.md) | `urn:ietf:params:jmap:cid` | SHA-256 digests on blob upload responses and FileNode objects |
| [draft-atwood-jmap-did-00](draft-atwood-jmap-did-00.md) | `urn:ietf:params:jmap:did` | Binds every JMAP account to a Decentralized Identifier; `did` property on types designated by consuming specs |
| [draft-atwood-jmap-object-uri-00](draft-atwood-jmap-object-uri-00.md) | — (URI scheme) | The `jmap:<type>:<accountId>:<objectId>` URI scheme for context-relative object references, and the JMAP Object URI Types registry |

---

## JMAP Chat Feature Summary

**Conversation types**
- Direct chats (one-to-one)
- Group chats (multi-party, with admin/member roles)
- Spaces: named workspaces with channels, role-based permissions, and category organization (what other systems call a server, workspace, or team)

**Messages**
- Plain text, Markdown, and a structured rich-text body format (`application/jmap-chat-rich`)
- File attachments with SHA-256 integrity verification
- @mentions as structured id lists, plus broadcast mentions (e.g. @here), separate from body formatting
- Message editing with revision history
- Soft deletion (tombstone) and hard deletion (sender-controlled expiry, burn-on-read)
- Reactions (emoji, including Space-scoped custom emoji)
- Threads (optional, server-advertised)
- Pinned messages and per-chat notification settings
- Out-of-band action invitations (video calls, payments, file links)

**Real-time**
- Typing indicators (ephemeral, best-effort, opt-out per chat)
- Presence: online/away/busy/invisible/offline with custom status text and emoji
- Read receipts with privacy opt-out (per-account and per-chat)
- WebSocket transport for low-latency bidirectional communication

**Push**
- Inline `ChatMessagePush` payloads, so mobile clients can show a notification without a follow-up request
- Mention-aware urgency (different priority for @mentions vs. regular messages)
- E2EE-aware: no body snippets for encrypted message bodies
- Fallback to standard `StateChange` under rate limiting

**Privacy and security**
- End-to-end encrypted relay deployments (server routes opaque ciphertext only)
- Federation requests signed with Ed25519 HTTP Message Signatures; optional DID-bound peer identity
- Receipt sharing opt-out (server suppresses outbound `Peer/receipt` calls)
- Typing indicator suppression per chat (`receiveTypingIndicators`)
- Contact blocking (messages and ephemeral events from blocked senders dropped server-side)
- SSRF-hardened blob fetch in federation

**Spaces**
- Role hierarchy with named permissions (view, send, pin, manage_channels, ban, etc.)
- Per-channel permission overrides
- Invite codes with expiry and use limits
- Bans with optional expiry
- Public and publicly-previewable visibility modes
- Optional shared file tree via JMAP FileNode

---

## Implementer Guides and Companion Documents

Non-normative. These explain *how* to implement and deploy the specs, and what each spec deliberately leaves open. When a guide conflicts with a draft, the draft is authoritative.

### Chat

| Document | For |
|---|---|
| [jmap-chat-implementer-guide.md](jmap-chat-implementer-guide.md) | Server and client implementers: governance, authorization, identity, and deployment-defined posture for the core spec |
| [jmap-chat-federation-guide.md](jmap-chat-federation-guide.md) | Operators: running federation as a service: peer auth, allowlists, abuse mitigation, observability |
| [jmap-push-platform-guide.md](jmap-push-platform-guide.md) | Server implementers: delivering any JMAP push to FCM, APNs, ADM, HPK, MiPush, WNS, and Web Push |
| [jmap-chat-push-platform-guide.md](jmap-chat-push-platform-guide.md) | Server implementers: encoding `ChatMessagePush` payloads per platform (supplement to the above) |
| [jmap-chat-wss-guide.md](jmap-chat-wss-guide.md) | WebSocket connection lifecycle, event handling, fan-out architecture |
| [jmap-chat-filenode-guide.md](jmap-chat-filenode-guide.md) | Operators: Space file storage at scale: backends, scanning, quotas, previews, grace period |
| [jmap-chat-calendars-guide.md](jmap-chat-calendars-guide.md) | Deployment posture for calendar binding, RSVP, availability, ICS parsing |
| [jmap-chat-tasks-guide.md](jmap-chat-tasks-guide.md) | Deployment posture for TaskList binding, Task↔Chat back-references, workflow |
| [jmap-chat-did-guide.md](jmap-chat-did-guide.md) | DID-bound federation authentication, resolution caching, DID document conventions, provisioning, recovery |
| [jmap-chat-vc-guide.md](jmap-chat-vc-guide.md) | Operators: gating Space creation, administration, and membership on Verifiable Credentials with the drafts as they stand |
| [jmap-chat-metadata-guide.md](jmap-chat-metadata-guide.md) | Using `draft-ietf-jmap-metadata` with JMAP Chat: namespace design and suggested keys |
| [jmap-chat-compliance-tests.md](jmap-chat-compliance-tests.md) | A test checklist and harness spec for verifying a JMAP Chat implementation, tagged by spec section and severity |

### VTC

| Document | For |
|---|---|
| [jmap-vtc-implementer-guide.md](jmap-vtc-implementer-guide.md) | Media stack, lifecycle, policy, and operational decisions the VTC spec leaves open |
| [jmap-vtc-e2ee-guide.md](jmap-vtc-e2ee-guide.md) | What real media stacks, SFUs, and gateways can do for E2EE today, and where the gaps are |
| [jmap-vtc-recording-guide.md](jmap-vtc-recording-guide.md) | Full VTCRecording lifecycle, consent, compliance, and storage decisions |
| [jmap-vtc-ring-push-guide.md](jmap-vtc-ring-push-guide.md) | Ring-call push notifications |
| [jmap-vtc-gateway-guide.md](jmap-vtc-gateway-guide.md) | PSTN, SIP, and H.323 gateway integration |
| [jmap-vtc-filenode-guide.md](jmap-vtc-filenode-guide.md) | Storing recordings and shared files in JMAP FileNode |
| [jmap-vtc-chat-integration-guide.md](jmap-vtc-chat-integration-guide.md) | Chat implementers adding calling |
| [jmap-vtc-chat-delegation-guide.md](jmap-vtc-chat-delegation-guide.md) | VTC implementers delegating in-call chat, reactions, captions, and lobby to JMAP Chat |

### Scene

| Document | For |
|---|---|
| [jmap-scene-implementer-guide.md](jmap-scene-implementer-guide.md) | Spatial state model, lifecycle, access control, and operational decisions the Scene spec leaves open |
| [jmap-scene-simulation-guide.md](jmap-scene-simulation-guide.md) | Real-time simulation layer: architecture, state reconciliation, authority models |
| [jmap-scene-wss-guide.md](jmap-scene-wss-guide.md) | Scene WebSocket subscriptions, rate limiting, visibility filtering, interop with Chat/VTC streams |
| [jmap-scene-vtc-integration-guide.md](jmap-scene-vtc-integration-guide.md) | Adding voice/video to Scene: spatial audio and UX patterns |
| [jmap-scene-2d-guide.md](jmap-scene-2d-guide.md) | 2D experiences on the 3D model: virtual offices, side-scrollers, tile grids, sprites |
| [jmap-scene-board-games-guide.md](jmap-scene-board-games-guide.md) | Board, card, and arcade games mapped onto Scene primitives |
| [jmap-scene-landscape-guide.md](jmap-scene-landscape-guide.md) | Comparison with prior spatial systems, protocols, and platforms |

### Shared building blocks and cross-cutting

| Document | For |
|---|---|
| [jmap-cid-guide.md](jmap-cid-guide.md) | CID integration patterns, encoding details, operational decisions |
| [jmap-did-guide.md](jmap-did-guide.md) | DID binding strategies, method tradeoffs, verification, lifecycle |
| [jmap-toip-mapping-guide.md](jmap-toip-mapping-guide.md) | DID/DIF/ToIP reviewers: maps the DID, Chat, Federation, Chat DID, and MIMI drafts onto the ToIP stack |
| [jmap-unified-client-architecture.md](jmap-unified-client-architecture.md) | Architecture for a single desktop client covering Mail, Chat, VTC, Scene, Push, FileNode, Calendars, Tasks, DID, and CID |

---

## Relationship to Other Standards

**Built on:** [RFC 8620](https://www.rfc-editor.org/rfc/rfc8620) (JMAP Core) and, for the WebSocket bindings, [RFC 8887](https://www.rfc-editor.org/rfc/rfc8887). All data types, methods, and push mechanisms are JMAP capabilities, so clients use the standard JMAP request/response framing.

**Analogous to:** [RFC 8621](https://www.rfc-editor.org/rfc/rfc8621) (JMAP for Mail). JMAP Chat is to instant messaging what JMAP Mail is to email.

**Complementary to:** [MIMI](https://datatracker.ietf.org/wg/mimi/about/) (More Instant Messaging Interoperability). MIMI targets provider-to-provider interoperability between large existing platforms; JMAP Chat targets decentralized mailbox-per-user federation and JMAP-native deployments. The two federation layers are separate, and `draft-atwood-jmap-chat-mimi-00` bridges them: it maps MIMI content to JMAP Chat Messages and lets a JMAP Chat server act as a MIMI provider.

**Optionally integrates with:**
- [JMAP FileNode](https://datatracker.ietf.org/doc/draft-ietf-jmap-filenode/): hierarchical file storage for Space file trees and VTC recordings
- [JMAP Blob Management](https://datatracker.ietf.org/doc/draft-ietf-jmap-blobext/): extended blob operations
- [MLS / RFC 9420](https://www.rfc-editor.org/rfc/rfc9420): end-to-end encryption key schedule for relay deployments
- [W3C DID Core](https://www.w3.org/TR/did-core/): the Decentralized Identifier data model used by the DID drafts
- [JMAP Quotas](https://datatracker.ietf.org/doc/draft-ietf-jmap-quotas/), [JMAP Metadata](https://datatracker.ietf.org/doc/draft-ietf-jmap-metadata/), [JMAP Calendars](https://datatracker.ietf.org/doc/draft-ietf-jmap-calendars/), [JMAP Tasks](https://datatracker.ietf.org/doc/draft-ietf-jmap-tasks/)

---

## Repository Layout

All files are Markdown in the repository root.

```
draft-atwood-jmap-*-00.md    Normative Internet-Drafts (kramdown-rfc source)
jmap-*-guide.md              Non-normative implementer and deployment guides
jmap-chat-compliance-tests.md
jmap-unified-client-architecture.md
PROJECT.md                   Project scope and standing editorial decisions
AGENTS.md                    Instructions for AI agents working in this repo
.beads/                      Issue tracker (bd)
```
