# JMAP Chat Metadata — Implementer's Guide

For server and client implementers considering `draft-ietf-jmap-metadata`
integration with `draft-atwood-jmap-chat-00`.
Covers namespace design, suggested keys, and deployment considerations.

Read `draft-ietf-jmap-metadata` (current at time of writing) and `draft-atwood-jmap-chat-00` first. This
guide does not re-state normative requirements; it covers what the specs leave
to implementations and offers patterns and starting points.

---

## A note on RFC 2119 keywords

This guide uses RFC 2119 keywords (MUST, SHOULD, MAY, REQUIRED, RECOMMENDED, etc.) for
clarity, but in the spirit of implementer guidance rather than as a normative protocol
specification:

- The drafts (`draft-atwood-jmap-chat-*.md`) are the normative source of truth. Where
  this guide describes a spec requirement using a keyword, the keyword reflects the
  spec's normativity; if guide and draft disagree, the draft wins.
- Where this guide uses a keyword for an operational practice, UX default, or deployment
  choice (e.g., "servers SHOULD log admin actions"), the keyword reflects implementer
  best-practice. Deviation does not affect protocol interop.
- Cite the spec, not the guide, when claiming normative authority.

---

## How to read this guide

`draft-ietf-jmap-metadata` defines per-type `metadata` (shared) and
`privateMetadata` (per-user) properties for data types listed in the
`urn:ietf:params:jmap:metadata` capability's `dataTypes` map. The chat spec
notes that servers MAY include `Chat`, `Message`, and `Space` in that map but
does not prescribe namespaces or keys.

This guide suggests concrete namespace layouts for each type. All suggested
keys use vendor (domain-name) namespaces so they can be adopted without IANA
registration. Deployments SHOULD define a single vendor namespace
(e.g., `chat.example.com`) and nest all keys under it to avoid proliferation.

Each section follows the same shape:

1. **Object type and property** — `metadata` (shared) or `privateMetadata` (per-user).
2. **Suggested keys** — with types and semantics.
3. **Use cases** — why a client or server would store this.
4. **Considerations** — pitfalls and interactions.

---

## 1. Chat privateMetadata

Private per-user annotations on a `Chat` object. These do not modify the
shared chat record and are invisible to other participants.

### Suggested keys

Under a vendor namespace (e.g., `chat.example.com`):

| Key | Type | Description |
|-----|------|-------------|
| `color` | `String|Object|Array` | Color value per the Color Representation convention in {{JMAP-CHAT}} (sRGB hex string, W3C Design Tokens object, or CAM16 coordinate array). |
| `category` | `String` | User-defined folder or grouping label. |
| `starred` | `Boolean` | Secondary "favorites" tier beyond server-side pinning. |
| `pinPosition` | `UnsignedInt` | Client-side sort key for pinned chats in the sidebar. |
| `snoozedUntil` | `UTCDate` | Hide the chat from default views until this time. |
| `draftReminder` | `UTCDate` | Nudge the user to reply after this time. |
| `notes` | `String` | Freeform scratchpad ("vendor chat for project X"). |
| `notificationSound` | `String` | URI or name of a per-chat notification tone. |
| `wallpaper` | `String` | `blobId` or URI for a per-chat background image. |

### Example

```json
{
  "privateMetadata": {
    "chat.example.com": {
      "color": "#e06c75",
      "category": "work/project-alpha",
      "starred": true,
      "pinPosition": 2,
      "notes": "Main vendor channel — ask for invoices here"
    }
  }
}
```

### Use cases

- **Color coding**: visual triage in a long chat list.
- **Snooze**: defer low-priority chats without muting.
- **Draft reminders**: surface a chat after a deadline passes.
- **Notes**: capture context that doesn't belong in the chat itself.

### Considerations

- `pinPosition` is purely client-side. The spec defines `pinnedMessageIds` for pinning messages within a chat. For pinning chats themselves in the sidebar, use private metadata (e.g., a `pinPosition` key). There is no server-side 'chat is pinned' indicator in the spec.
- `snoozedUntil` requires the client to re-evaluate on each sync. Servers
  do not enforce snooze; it is a UI hint.
- `wallpaper` referencing a `blobId` depends on the blob remaining available.
  Clients SHOULD fall back gracefully if the blob is garbage-collected.

---

## 2. Message privateMetadata

Private per-user annotations on a `Message` object.

### Suggested keys

| Key | Type | Description |
|-----|------|-------------|
| `bookmark` | `Boolean` | Saved for later / flagged. |
| `highlightColor` | `String` | Marker-pen overlay color. |
| `tags` | `String[]` | Personal labels: `"receipt"`, `"actionable"`, `"important"`. |
| `readLaterPosition` | `UnsignedInt` | Sort key in a "read later" queue. |
| `followUpAt` | `UTCDate` | Surface the message again at this time. |
| `linkedTask` | `String` | URI or ID pointing to an external task tracker item. |
| `summary` | `String` | Client-generated or AI-generated one-line summary. |
| `translation` | `Object` | Cached translation: `{ "lang": "es", "text": "..." }`. |

### Example

```json
{
  "privateMetadata": {
    "chat.example.com": {
      "bookmark": true,
      "tags": ["receipt", "tax-2025"],
      "followUpAt": "2026-07-01T09:00:00Z",
      "summary": "Vendor confirmed delivery by June 15"
    }
  }
}
```

### Use cases

- **Bookmarks**: personal "saved messages" view across all chats.
- **Tags**: faceted search/filter without server-side label objects.
- **Follow-up**: time-shifted reminders tied to a specific message.
- **Translation cache**: avoid re-translating on every view; store the result
  once and display it inline.
- **Linked tasks**: bridge between chat and project management without
  requiring the `draft-atwood-jmap-chat-tasks-00` extension.

### Considerations

- Message metadata scales with message volume. Servers MAY impose per-type
  quota via `DataTypeMetadataInfo` to prevent unbounded growth.
- `translation` objects can be large. Servers with a `maxDepth` of 1 would
  reject nested objects; clients SHOULD check the capability before writing
  nested values.
- `followUpAt` is a client-side hint. Unlike Chat-level `snoozedUntil`, it
  does not hide the message — it adds a reminder surface.

---

## 3. Space privateMetadata

Private per-user annotations on a `Space` object.

### Suggested keys

| Key | Type | Description |
|-----|------|-------------|
| `colorTag` | `String|Object|Array` | Color value per the Color Representation convention in {{JMAP-CHAT}} (sRGB hex string, W3C Design Tokens object, or CAM16 coordinate array). |
| `sidebarPosition` | `UnsignedInt` | Client-side sort key for sidebar ordering. |
| `collapsed` | `Boolean` | Whether the Space's channel tree is collapsed in the UI. |
| `nickname` | `String` | Personal rename of a Space without changing the shared name. |
| `lastVisitedChannelId` | `Id` | Channel to restore focus on re-open. |
| `mutedChannelIds` | `Id[]` | Channels the user has locally muted (if not server-modeled). |
| `notes` | `String` | Freeform scratchpad ("onboarding server — leave after 30 days"). |
| `customIcon` | `String` | `blobId` for a personal override of the Space avatar. |

### Example

```json
{
  "privateMetadata": {
    "chat.example.com": {
      "colorTag": "#61afef",
      "sidebarPosition": 1,
      "collapsed": false,
      "nickname": "Backend Team",
      "lastVisitedChannelId": "ch-abc123"
    }
  }
}
```

### Use cases

- **Sidebar ordering**: user-defined Space sort independent of server-side
  ordering or alphabetical sort.
- **Collapse state**: persist UI state across devices via JMAP sync.
- **Nickname**: personalize Space names without affecting other members.
- **Last visited channel**: restore context on re-open without server support.

### Considerations

- `mutedChannelIds` overlaps with server-side mute if the deployment models
  mute as a first-class Chat or Space property. Prefer the server-side model
  when available; use metadata only as a fallback for deployments that lack it.
- `lastVisitedChannelId` references a channel that may be deleted or made
  inaccessible. Clients SHOULD fall back to the Space's default channel.
- `collapsed` and `sidebarPosition` are UI state. Syncing UI state via
  metadata is convenient for multi-device consistency but can cause surprises
  if the user expects per-device layout. Clients MAY offer a "sync UI
  preferences" toggle.

---

## 4. Chat metadata (shared)

Shared annotations on a `Chat` visible to all participants with read access.
Use sparingly — most per-user preferences belong in `privateMetadata`.

### Suggested keys

| Key | Type | Description |
|-----|------|-------------|
| `integrationStatus` | `Object` | Status from an external integration: `{ "type": "jira", "issueKey": "PROJ-42", "status": "open" }`. |
| `summary` | `String` | Pinned summary or topic elaboration beyond the Chat's `topic` field. |
| `externalLink` | `String` | URI linking the chat to an external resource (wiki page, project board). |

### Considerations

- Shared metadata is visible to all readers. Do not store anything sensitive
  or user-specific here.
- Write access to shared metadata follows the same ACL as the parent object's
  `/set`. For direct chats, the owner may write freely. For group chats, servers SHOULD restrict shared metadata writes to admins (consistent with other shared-state mutations like `name` and `description`). For channel chats, servers SHOULD restrict writes to members with appropriate Space permissions (e.g., `"manage_channels"`).
- Integration metadata (`integrationStatus`) is a natural fit for bot-written
  annotations. Bots SHOULD use a vendor namespace scoped to the integration
  (e.g., `jira.example.com`) to avoid collisions with user-facing namespaces.

---

## 5. Message metadata (shared)

Shared annotations on a `Message` visible to all participants.

### Suggested keys

| Key | Type | Description |
|-----|------|-------------|
| `linkPreview` | `Object` | Cached link preview: `{ "url": "...", "title": "...", "description": "...", "imageUrl": "..." }`. |
| `moderationNote` | `String` | Note from a moderator explaining an edit or action on the message. |

### Considerations

- `linkPreview` avoids every client independently fetching the same URL.
  The sending client or a server-side bot writes the preview once; all
  readers benefit.
- `moderationNote` makes moderation actions transparent. This complements
  the server-set `editedAt` and `deletedAt` fields on Message.

---

## 6. Space metadata (shared)

Shared annotations on a `Space` visible to all members.

### Suggested keys

| Key | Type | Description |
|-----|------|-------------|
| `welcomeMessage` | `String` | Displayed to new members on join. |
| `guidelines` | `String` | Community guidelines or rules (Markdown). |
| `externalLinks` | `Object[]` | Array of `{ "label": "...", "url": "..." }` for sidebar links. |

### Considerations

- `welcomeMessage` and `guidelines` overlap with Space `topic` and
  `description`. Use metadata when the content is auxiliary and you do not
  want to overload the core fields.
- Write access to shared Space metadata should be restricted to roles with
  `manage_space` permission in practice, even though the metadata spec itself
  does not define role-based access. Servers SHOULD enforce this at the `/set`
  level.

---

## 7. Cross-cutting considerations

### Namespace hygiene

- Use one vendor namespace per organization or product. Nesting keys under a
  single namespace keeps the metadata object shallow and predictable.
- Avoid registered (dot-free) namespaces until they are formally defined by
  an IETF specification. Vendor namespaces require no registration.

### Quota and size

- `draft-ietf-jmap-metadata` allows servers to enforce per-type quota.
  Clients SHOULD handle `overQuota` SetErrors gracefully.
- Message-level metadata scales with message count. A deployment with
  millions of messages should set conservative per-type limits or restrict
  Message metadata to `privateMetadata` only.

### Multi-device sync

- Metadata properties flow through standard `/get` and `/changes`, so
  multi-device sync is automatic. Clients that store UI state in metadata
  (collapse, sidebar position) should be aware that changes propagate to all
  devices.
- Conflict resolution follows normal JMAP `/set` semantics: last writer wins
  at the property-patch level. For frequently-updated keys like
  `lastVisitedChannelId`, this is acceptable; for keys where merge semantics
  matter (e.g., `tags` arrays), clients should read-before-write.

### Migration from custom storage

- Deployments that currently store client preferences in local storage,
  cookies, or a proprietary side-channel can migrate to JMAP Metadata for
  cross-device consistency. The migration path is:
  1. Advertise the data type in `dataTypes`.
  2. On first sync, client reads `privateMetadata`; if empty, writes current
     local preferences into it.
  3. On subsequent syncs, `privateMetadata` is the source of truth.
