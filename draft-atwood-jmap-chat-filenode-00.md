---
title: JMAP Chat File Storage
abbrev: JMAP Chat FileNode
docname: draft-atwood-jmap-chat-filenode-00
category: std
stream: ietf

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
  JMAP-CHAT:
    title: JMAP for Chat
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-00
    date: 2026
  JMAP-FILENODE:
    title: JMAP File Storage Extension
    author:
      fullname: Bron Gondwana
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-filenode-13
    date: 2026
    target: https://datatracker.ietf.org/doc/draft-ietf-jmap-filenode/

informative:
  JMAP-CHAT-FED:
    title: JMAP Chat Federation
    target: https://datatracker.ietf.org/doc/draft-atwood-jmap-chat-federation/
  JMAP-BLOBEXT:
    title: JMAP Blob Management Extensions
    target: https://datatracker.ietf.org/doc/draft-ietf-jmap-blobext/
  RFC9425:

--- abstract

This document defines JMAP Chat File Storage, a companion specification to JMAP Chat ({{JMAP-CHAT}}) and the JMAP File Storage extension ({{JMAP-FILENODE}}). It specifies how a Space in JMAP Chat is associated with a hierarchical file tree managed by {{JMAP-FILENODE}}, how Space permissions govern access to that tree, how message attachments may optionally link to FileNode entries, and how the file tree is treated when a Space is destroyed.

--- middle

# Introduction

{{JMAP-CHAT}} defines Spaces as named containers for channel conversations, members, and roles — analogous to what other systems call workspaces, servers, or teams. A common feature of such platforms is a shared file storage area where members can upload, browse, and download files independently of individual messages.

{{JMAP-FILENODE}} defines a JMAP extension for hierarchical file storage, surfacing blobs as a navigable filesystem with metadata including names, timestamps, and media types. This document binds those two extensions: a Space may have an associated FileNode tree that is governed by the Space's membership and role permissions, giving members a shared file library alongside their channels.

{{JMAP-CHAT}} includes a note in the Space section that "a future companion specification could define Space-scoped file storage by associating a Filenode namespace with each Space." This document is that specification.

## Design Principles

This specification is intentionally narrow:

- File storage is defined for Spaces only. Group chats and direct chats are out of scope.
- The Space object is not modified. The association between a Space and its file tree is discovered through the FileNode role mechanism defined in {{JMAP-FILENODE}}, not through a new field on the Space.
- The permission model maps Space roles to FileNode access in a flat table. Per-channel FileNode ACL overrides are not defined.
- Message attachment integration is opt-in. Attachments are not automatically mirrored into the file tree.
- The upload and download flow is entirely that of {{JMAP-FILENODE}}, without modification.
- Federation semantics are out of scope for this version.

## Relationship to JMAP FileNode

This document defines a profile of {{JMAP-FILENODE}} for use with JMAP Chat Spaces. It registers a new well-known FileNode role, defines a Space-to-FileNode permission mapping, and adds optional fields to the JMAP Chat Attachment type. It does not redefine any FileNode methods, data types, or error codes.

Implementations MUST support {{JMAP-FILENODE}} as a prerequisite for implementing this document. Implementations MUST also support {{JMAP-CHAT}} as a prerequisite.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Terminology from {{RFC8620}}, {{JMAP-CHAT}}, and {{JMAP-FILENODE}} is used throughout.

This specification follows the URN convention established in {{JMAP-CHAT}}: capability identifiers (strings appearing in the JMAP Session `capabilities` object) use the `urn:ietf:params:jmap:` namespace, while data-model value strings (strings appearing inside object fields, such as role values and endpoint type URIs) use the `urn:jmap:` namespace.

The following term is specific to this document:

Space file tree:
: The FileNode subtree associated with a given Space, rooted at the FileNode whose `role` is `"urn:jmap:chat:filenode:space"` and whose `spaceId` matches the Space's `id`.

# Capability {#capability}

Servers supporting this specification advertise the `urn:ietf:params:jmap:chat:filenode` capability in the JMAP Session object. This capability is meaningful only when both `urn:ietf:params:jmap:chat` and `urn:ietf:params:jmap:filenode` are also present.

## Session-Level Capability Object

The value of `capabilities["urn:ietf:params:jmap:chat:filenode"]` at the session level is an empty object `{}`.

## Account-Level Capability Object

The value of `accountCapabilities["urn:ietf:params:jmap:chat:filenode"]` is a JSON object with the following fields:

`maxSpaceStorageBytes` (UnsignedInt, optional):
: Maximum total blob storage in bytes across all FileNodes in a single Space's file tree. If absent, the server does not advertise a per-Space limit; limits from the JMAP Quota extension {{RFC9425}} MAY apply. When a `FileNode/set` create would cause the Space file tree to exceed this limit, the server MUST return an `overQuota` SetError (as defined in {{RFC8620}} Section 5.3).

`autoCreateFileTree` (Boolean):
: When `true`, the server automatically creates a Space file tree root for each new Space at Space creation time and includes the new FileNode in subsequent `FileNode/changes` responses. When `false`, no file tree is created until a client or administrator explicitly does so via `FileNode/set`. Default is `false`. This field is REQUIRED in the account-level capability object when `urn:ietf:params:jmap:chat:filenode` is advertised.

# Space File Tree {#space-file-tree}

A Space's file tree is rooted at a single FileNode identified by the well-known `role` value `"urn:jmap:chat:filenode:space"` and the `spaceId` extension property defined below.

## The `spaceId` Extension Property {#filenode-extension}

This document defines the following extension property on FileNode objects that serve as Space file tree roots (i.e., where `role` is `"urn:jmap:chat:filenode:space"`):

`spaceId` (String, immutable):
: The `id` of the JMAP Chat Space this FileNode is associated with. Supplied by the client at creation and treated as immutable thereafter. Absent on all FileNode objects that are not Space file tree roots.

Servers MUST support `spaceId` as a filter criterion in `FileNode/query`, enabling clients to efficiently locate the root for a given Space.

## Creating the File Tree Root {#root-creation}

When a client creates a FileNode with `role: "urn:jmap:chat:filenode:space"`, it MUST supply `spaceId`. The server MUST validate:

- The Space identified by `spaceId` exists and the requesting user is a member. If not, the server MUST return `forbidden`. Servers SHOULD return `forbidden` in both cases — Space not found and user not a member — to avoid revealing Space existence to non-members.
- No file tree root already exists for that Space (at most one root per Space is permitted). If one exists, the server MUST return `invalidArguments`.
- The requesting user holds sufficient permission to create the root (see {{permissions}}). If not, the server MUST return `forbidden`.

On any of these failures, the server MUST NOT create the FileNode.

## Discovering the File Tree Root {#discovery}

To locate the file tree root for a given Space, clients issue a `FileNode/query` request filtering on `role: "urn:jmap:chat:filenode:space"` and `spaceId: <id>`. When a file tree exists for the Space, exactly one matching FileNode is returned. When no file tree exists, the result is empty; clients MAY attempt to create one via `FileNode/set` (subject to the rules in {{root-creation}}), or MAY present the absence of a file tree as an informational state to the user.

Clients MAY cache the file tree root `id` for the duration of the session and re-query on `cannotCalculateChanges`.

FileNode/changes responses MUST be scoped to file trees the requesting user has access to. A user who is not a member of a Space MUST NOT receive change notifications for that Space's file tree.

# Permission Mapping {#permissions}

Access to a Space's file tree is governed entirely by the requesting user's Space membership and role permissions, as defined in {{JMAP-CHAT}}. Space `ChannelPermission` per-channel overrides do not apply to file tree access.

Users who are not members of the Space — including authenticated users on the same server — MUST NOT be granted access to the Space file tree, regardless of any FileNode-level ACL entries that may exist. An exception applies during any grace period defined in {{deletion}}: former members retain the access level they held immediately before the Space was destroyed until the grace period expires.

For Space members, the following table maps role permissions to FileNode access. Permissions are cumulative: a member holding multiple roles receives the union of all applicable grants.

| Space condition | FileNode access granted |
|---|---|
| Member with `"view"` | Read: list directories, read file content |
| Member with `"send"` | Read + Create: upload new files and directories |
| Member with `"manage_channels"` | Read + Create + Delete: remove any file or directory |
| Member with `"manage_space"` | Read + Create + Delete + Administer: manage ACLs, rename the tree root |

Note: Space-scoped ChannelPermission per-channel overrides do not restrict file-tree access. A member who has been denied `"view"` on a specific channel via ChannelPermission can still read the Space's file tree, because file-tree access is governed by Space-level permissions only. Deployments requiring per-channel file isolation SHOULD use separate Spaces.

Creating the Space file tree root (a FileNode with `role: "urn:jmap:chat:filenode:space"`) requires `"manage_space"` permission. This is a higher bar than the general Create access granted by `"send"`, because establishing the root is a one-time structural operation that affects all Space members.

The server MUST enforce this mapping on every FileNode operation, deriving the requesting user's membership and roles from its own authoritative state. Clients MUST NOT be trusted to assert their own access level. When a Space member's roles change, the server SHOULD reflect the updated FileNode access immediately, without requiring any client action.

# Attachment Link {#attachment-link}

The `Attachment` type defined in {{JMAP-CHAT}} is extended with the following optional fields when the `urn:ietf:params:jmap:chat:filenode` capability is active.

## filenodeId {#filenode-field}

`filenodeId` (String, optional):
: The `id` of a FileNode in the Space's file tree. When present in a `Message/set` create request for a channel Chat (a Chat with `kind: "channel"` as defined in {{JMAP-CHAT}}), the server uses that FileNode's blob as the attachment blob. When `filenodeId` is present on an Attachment, the fields `blobId`, `filename`, `contentType`, `size`, and `sha256` become OPTIONAL for that Attachment entry; the server MUST populate any absent fields from the referenced FileNode's properties before storage. If `filenodeId` is absent, these fields retain their normal required status per {{JMAP-CHAT}}. If the client supplies these fields, the server MAY validate them against the FileNode and MAY reject the request with `invalidArguments` if they conflict.

When `filenodeId` is present, the server MUST verify that the referenced FileNode exists and belongs to the same Space. If validation fails, the server MUST return a SetError of type `invalidProperties` naming `filenodeId`. Additionally, the server MUST verify:

- The sender has at least read access to the FileNode under the mapping in {{permissions}}.

The FileNode is not modified when linked as an attachment. The message's attachment blob is stored by reference at send time. Subsequent modification or deletion of the FileNode by any member does not affect the stored message.

## thumbnailBlobId {#thumbnail}

`thumbnailBlobId` (String, optional):
: The `blobId` of a thumbnail or preview image derived from the FileNode's blob. SHOULD be set only when `filenodeId` is also present. Receiving clients MAY display this blob as an inline preview without fetching the full attachment. Servers MUST store and return `thumbnailBlobId` without modification; servers MUST NOT silently substitute a server-generated thumbnail for a client-supplied value.

When a FileNode contains an image blob, clients MAY generate a thumbnail for inline preview using `Blob/convert` as defined in {{JMAP-BLOBEXT}}, provided the server advertises the `urn:ietf:params:jmap:blob2` capability. The workflow is:

1. The client calls `Blob/convert` on the FileNode's blob, specifying a target `type` suitable for preview display (for example, `image/jpeg` or `image/webp`) and any implementation-defined size or quality parameters.
2. The server returns a new `blobId` for the converted thumbnail.
3. The client includes this `blobId` as `thumbnailBlobId` in the Attachment alongside `filenodeId`.

Clients whose server does not advertise `urn:ietf:params:jmap:blob2` SHOULD omit `thumbnailBlobId`. Receiving clients SHOULD render the full attachment blob when `thumbnailBlobId` is absent.

## Federation Hint {#attachment-federation}

Clients MAY include `filenodeId` in outbound `Peer/deliver` calls as an informational hint. Receiving servers that do not support this extension MAY ignore it. Receiving servers that do support this extension MAY use the `filenodeId` to present the attachment as a file reference — for example, displaying a file name and a link to browse or download the file — rather than treating it solely as an opaque blob copy.

# Space Deletion {#deletion}

When a Space is destroyed via `Space/set` destroy, the server SHOULD also destroy the associated file tree and all FileNodes within it.

The server MAY instead retain the file tree for an implementation-defined grace period, during which former Space members MAY continue to access files they had access to before the Space was destroyed. After the grace period, the server SHOULD destroy the file tree. Servers SHOULD NOT leave file trees permanently accessible after Space destruction, as orphaned trees with no governing membership model create access control ambiguity.

Once the grace period expires, the server MUST return `forbidden` on any FileNode operation attempted by a former Space member. Servers SHOULD emit a `FileNode` state change event at the moment the file tree is destroyed, allowing clients to detect the transition without polling.

Clients SHOULD treat the file tree as potentially inaccessible once they receive confirmation that its associated Space has been destroyed.

# Security Considerations {#security}

## Access Control Enforcement

The permission mapping in {{permissions}} MUST be enforced server-side on every FileNode operation. The server MUST derive the requesting user's Space membership and roles from its own authoritative state, not from any client-supplied argument.

A user removed from a Space loses FileNode access to that Space's file tree. The server SHOULD deny subsequent FileNode operations from that user immediately upon membership removal, independent of any grace period that may apply to Space destruction ({{deletion}}).

## Attachment filenodeId Validation

The `filenodeId` field is client-supplied and MUST be treated as untrusted. The server SHOULD verify that the referenced FileNode belongs to the Space file tree for the Space containing the message's channel Chat. Servers SHOULD NOT permit a `filenodeId` referencing a FileNode from a different Space or from outside any Space file tree, as this could allow cross-Space data exfiltration through the message attachment mechanism.

## File Content

Servers do not inspect the content of files stored in the file tree beyond blob integrity checks (size, sha256). Implementations that apply content scanning SHOULD do so at upload time, before the FileNode is made visible to other members.

## Federation

This specification does not define file tree access in federated (mailbox-per-user) deployments as described in {{JMAP-CHAT-FED}}. Implementations operating in a federated context SHOULD restrict file tree access to the local server's own account holders. A future companion document MAY define federation semantics for Space file storage.

# IANA Considerations {#iana}

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP Capabilities" registry:

Capability Name:
: `urn:ietf:params:jmap:chat:filenode`

Intended Use:
: common

Change Controller:
: IETF

Specification document:
: This document.

Security and privacy considerations:
: See {{security}} of this document.

## JMAP FileNode Roles Registry

IANA is requested to register the following entry in the "JMAP FileNode Roles" registry defined by {{JMAP-FILENODE}}:

Role:
: `urn:jmap:chat:filenode:space`

Description:
: Root directory of the file storage tree associated with a JMAP Chat Space. At most one FileNode with this role exists per Space (when file storage is enabled for that Space). The associated Space is identified by the `spaceId` extension property defined in this document.

Reference:
: This document.

--- back

# Acknowledgements

The author thanks Bron Gondwana for {{JMAP-FILENODE}}, the file storage extension this specification profiles, and the JMAP working group for {{RFC8620}}.
