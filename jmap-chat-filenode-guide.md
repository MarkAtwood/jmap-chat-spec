# JMAP Chat File Storage — Implementer's Guide

For operators of mailbox servers offering JMAP Chat Space file storage per
`draft-atwood-jmap-chat-filenode-00`. Covers the operational and policy
posture decisions that the filenode draft deliberately leaves open.

Read the draft and `draft-ietf-jmap-filenode` first. This guide does not
re-state normative requirements; it covers what the spec leaves to
deployments and offers patterns and starting points for running file
storage as a service.

The filenode draft itself defines the wire surface: the
`urn:ietf:params:jmap:chat:filenode` capability, the
`urn:jmap:chat:filenode:space` FileNode role, the `spaceId` extension
property, the Space-role-to-FileNode-access permission mapping, the
attachment-link fields (`filenodeId`, `thumbnailBlobId`), and the
Space-destruction grace-period contract. The main
`jmap-chat-implementer-guide.md` covers core deployment topics
(governance, authorization, identity, storage of chat content). This
guide focuses on what is unique to running file storage at scale.

---

## A note on RFC 2119 keywords

This guide uses RFC 2119 keywords (MUST, SHOULD, MAY, REQUIRED, RECOMMENDED, etc.) for
clarity, but in the spirit of implementer guidance rather than as a normative protocol
specification:

- The drafts (`draft-atwood-jmap-chat-*.md`) are the normative source of truth. Where
  this guide describes a spec requirement using a keyword, the keyword reflects the
  spec's normativity; if guide and draft disagree, the draft wins.
- Where this guide uses a keyword for an operational practice, deployment policy, or
  default value choice (e.g., "operators SHOULD scan uploaded files for malware"), the
  keyword reflects implementer best-practice. Deviation does not affect protocol
  interop.
- Cite the spec, not the guide, when claiming normative authority.

---

## How to read this guide

`draft-atwood-jmap-chat-filenode-00` deliberately specifies a minimal
wire contract (one new capability, one well-known FileNode role, one
extension property, two attachment fields, one permission mapping) and
defers backend choice, content policy, malware scanning, quota
management, preview generation, retention mechanism, performance
tuning, and migration patterns to operators. File storage deployment
models range from "single-server local disk for a small team" to
"globally-replicated S3-compatible object storage with CDN and
dedicated thumbnail-rendering tier". The spec cannot predict which
model fits any one deployment; this guide helps operators think through
the choices.

Each section follows the same shape as the broader
`jmap-chat-implementer-guide.md` and the other companion guides:

1. **What the spec leaves open** — with a citation.
2. **What you must decide.**
3. **Considerations.**
4. **Common patterns** from production file-storage systems.
5. **Recommended starting point** — defensible default, not normative.

---

## 1. Storage backend choice

### What the spec leaves open

The filenode draft does not constrain how blob storage is backed.
`draft-ietf-jmap-filenode` defines the abstract `FileNode` data model
and the `Blob/*` interactions; the underlying bytes live somewhere
implementation-defined. Local filesystem, S3-compatible object storage,
a dedicated file server, or hybrid arrangements are all conformant.

### What you must decide

- Which storage backend (or which combination of backends) underlies
  blob storage.
- How blob metadata (size, sha256, MIME type) is indexed for fast
  lookup and `FileNode/query` evaluation.
- Whether blobs are stored co-located with chat content (one database,
  one storage tier) or in a separate storage system.
- How the backend's durability and replication properties map to the
  deployment's SLO for file storage.
- Whether to use a content-addressable scheme (deduplicate identical
  blobs across Spaces) or per-FileNode separate storage.

### Considerations

- *Local filesystem* is the simplest backend: blobs on disk, FileNode
  metadata in a database, no external dependencies. Operationally
  trivial; does not scale beyond one server's local storage; backup
  becomes the bottleneck. Best fit for small single-server deployments.
- *S3-compatible object storage* (Amazon S3, Google Cloud Storage,
  Azure Blob Storage, MinIO, Ceph RGW, Backblaze B2) decouples storage
  capacity from server count. Durability and replication are managed by
  the storage provider; the server holds metadata only. Adds an
  external dependency and per-request latency; the operational model
  is well-understood at scale.
- *Dedicated file server* (NFS, SMB, a purpose-built file-storage tier)
  is a middle ground: shared storage across server nodes without the
  complexity of object storage. Operationally heavy compared to object
  storage; legacy choice for many enterprise deployments.
- *Hybrid* arrangements (recent / hot blobs on local SSD; older blobs
  archived to object storage) optimize for cost without sacrificing
  retrieval latency for active files. Requires a tiering policy and a
  migration mechanism; useful at scale.
- *Content-addressable storage* (blob-by-sha256) lets multiple Spaces
  share storage for identical files (e.g., a widely-circulated meme,
  a corporate logo embedded across teams). Reduces storage cost; adds
  complexity around per-blob access counting for safe deletion.
- *Blob metadata* (size, sha256, MIME type, FileNode parent pointer)
  belongs in an indexed metadata store, regardless of where the bytes
  live. The metadata store is the source of truth for `FileNode/query`
  and the access-control checks; the bytes are fetched by reference.
- *Durability SLO mapping*: a 99.999999999% durability claim from an
  object-storage provider does not survive misconfiguration. Test
  backup-and-restore procedures on a regular schedule regardless of
  backend.

### Common patterns

| System | Storage backend |
|---|---|
| Slack (workspace files) | S3 with metadata in a relational DB |
| Discord (CDN-served attachments) | Object storage (Cloudflare R2 era / S3 prior) with CDN edge cache |
| Mattermost (self-host) | Local filesystem or S3-compatible; deployment-time choice |
| Element / Matrix file storage | Filesystem-backed media repo; object-storage variants |
| Nextcloud | Pluggable backends; default is filesystem |

### Recommended starting point

For deployments with under approximately 100 active users: **local
filesystem** with daily snapshot backups. Simplest; lowest operational
overhead.

For deployments above that scale, or those expecting growth: **S3-compatible
object storage** with a relational or document-store metadata index.
The decoupling of capacity from server count is the right shape for
scaling. Use a per-Space (or per-deployment, per the per-Space quota
defined in the filenode draft's `maxSpaceStorageBytes`) prefix
convention in the object-storage namespace to keep operational
inspection simple (e.g., `space/<spaceId>/<filenodeId>` keys).

Deduplication by sha256 is a deferred optimization; do not start with
it. Add it once measured duplication justifies the operational
complexity.

Whatever backend is chosen, document its durability claims, its backup
schedule, and its tested recovery point objective (RPO) in the operator
handbook. File loss is one of the most user-visible failure modes;
operators need clarity about the SLO.

---

## 2. File type policy

### What the spec leaves open

The filenode draft and `draft-ietf-jmap-filenode` do not constrain
which file types may be uploaded. Servers receive blobs with
client-supplied MIME types; whether to accept, reject, or transform
based on type is operator policy. The filenode draft's
Security Considerations section (draft §Security Considerations), "File Content" subsection notes only that scanning is at
operator discretion.

### What you must decide

- Allowlist (accept only specified MIME types), denylist (reject
  specified MIME types), or open (accept all).
- MIME-type verification: trust the client's `Content-Type` header, or
  re-derive from magic bytes.
- Handling of executables (Windows `.exe`, Linux ELF, macOS Mach-O,
  scripts).
- Archive expansion: scan inside `.zip`, `.tar`, `.tar.gz`, etc., or
  treat the archive as opaque.
- Office documents with macro capability (`.docm`, `.xlsm`).
- Image and media format support (which formats trigger preview
  generation; which are rejected outright).
- Maximum file size (interacts with per-Space quotas; see §4).

### Considerations

- *Allowlist* is the safest model: only known-acceptable types pass.
  Operationally tight; new file types require explicit allowlist
  expansion. Best fit for regulated or high-security deployments.
- *Denylist* is the most permissive useful model: accept anything not
  on the bad-types list. Easier on users; harder to keep current as new
  attack types emerge.
- *Open* (no type policy) is acceptable for purely internal
  deployments where every uploader is trusted; otherwise risky.
- *Client `Content-Type` is untrusted*: a client claiming `image/png`
  may be uploading something else. Re-derive the MIME type from the
  blob's magic bytes (file signature) for any policy decision.
  Libraries: `libmagic`, `file(1)`'s logic ported to most languages.
- *Executables* are the highest-risk category. A `.exe` uploaded to a
  chat Space file tree is unlikely to be benign. Most consumer
  deployments either reject executables outright, quarantine them
  (require admin review before they are downloadable), or limit to
  uploaders with elevated permissions.
- *Archive expansion* (zip-bomb defense): a 42 KB nested zip can
  expand to 4 PB. If you scan inside archives, implement strict expansion
  limits (size cap, depth cap, file-count cap). If you do not scan
  inside archives, you cannot apply per-file-type policy to their
  contents — the archive is a policy bypass.
- *Office macros*: `.docm`, `.xlsm`, and similar formats can carry
  executable code. Treat as elevated-risk types; some deployments
  silently strip the macro layer; others reject the file outright.
- *Image formats*: SVG can carry script (XSS via `<script>` tags);
  treat differently from raster images. WebP, HEIC, and AVIF have
  decoder vulnerabilities historically; ensure decoders are current.
- *Maximum file size*: interacts with the filenode draft's
  `maxSpaceStorageBytes` (per-Space quota; see §4). A single-file cap
  bounds the maximum impact of one upload; a per-Space cap bounds total
  growth.

### Common patterns

| System | File type policy |
|---|---|
| Slack | Open with maximum size limits per plan tier; warns on executables |
| Discord | Open with maximum size limits; some types blocked at CDN edge |
| Microsoft Teams | Allowlist with admin-configurable types; macro-enabled office docs treated as elevated risk |
| Email attachments | Variable; many MTAs reject by extension regardless of MIME type |
| Enterprise file servers | Allowlist driven by data-classification policy |

### Recommended starting point

For consumer / social deployments: **denylist with magic-byte
verification**. Block known-dangerous extensions (`.exe`, `.scr`,
`.bat`, `.cmd`, `.com`, `.cpl`, `.dll`, `.jar`, `.js`, `.ps1`, `.sh`,
`.vbs`, `.wsh`) by both extension and magic-byte signature. Apply a
file-size cap (e.g., 100 MB per file as a default; adjust based on
deployment posture).

For enterprise deployments: **allowlist** anchored on the
deployment's data-classification policy. Common allowlists include
office documents, common image formats, PDFs, and plain text. Macro-enabled
office formats are treated as elevated risk: either rejected,
quarantined, or marked for admin review.

For both: **re-derive MIME type from magic bytes** for any policy
decision. Do not trust client-supplied `Content-Type`.

Archive handling: deployments SHOULD NOT recursively expand archives by
default. If they do, apply strict expansion limits (size cap 100 MB,
depth cap 3, file-count cap 1000) and abort on limit hit. Archive
contents are not scanned in default policy; treat the archive as
opaque.

Document the file-type policy in the deployment's user-facing
documentation. Users should know what they can upload.

---

## 3. Virus / malware scanning

### What the spec leaves open

The filenode draft (draft §Security Considerations) "File Content" subsection states only
that operators MAY apply scanning at upload time. No specific scanner,
timing, or action-on-detection is mandated. The wire protocol does not
carry scan results; clients have no formal channel for "this file was
quarantined" responses beyond standard FileNode errors.

### What you must decide

- Whether to scan at all.
- When to scan: upload-time (synchronous; user waits), upload-time
  (asynchronous; quarantined until cleared), download-time (each
  retrieval re-scans), periodic background (catches latent-signature
  detections).
- Which scanning engine(s): open-source (ClamAV), commercial
  (Bitdefender, Sophos, etc.), cloud APIs (VirusTotal, Microsoft
  Defender for Storage), in-house heuristics.
- Action on detection: block (refuse the upload), quarantine (accept
  but make undownloadable), flag (allow but mark with a warning),
  notify uploader, notify recipients, alert operator.
- Disposition of clean files: store the scan result for future
  reference, or re-scan on download.
- How false positives are handled (uploader appeals process).

### Considerations

- *No scanning* is acceptable only for purely internal trusted
  deployments. Any deployment accepting uploads from a non-trivially-trusted
  user population should scan.
- *Upload-time synchronous*: the user waits for scan completion before
  the file is visible. Latency-sensitive; user-visible delay on every
  upload. Best for small files and fast scanners; impractical for large
  files or slow signature-based scanners.
- *Upload-time asynchronous*: the file is accepted into a quarantine
  state; recipients see "scanning in progress" until cleared.
  Better UX than synchronous; requires a scan-status field and client
  rendering for the in-progress state.
- *Download-time*: each retrieval triggers a scan. Catches signature
  updates that detect previously-clean files; multiplies scan cost by
  download count; high latency per download.
- *Periodic background*: re-scan stored files on a schedule (e.g.,
  weekly). Catches latent detections without per-download cost. Does
  not block uploads or downloads; lower urgency than upload-time
  scanning for new uploads.
- *Hybrid* (upload-time synchronous for small / fast-scanned files +
  background re-scan + signature-update-triggered re-scan) is
  common at scale.
- *Scanning engine choice*: ClamAV is the open-source default; good
  enough for many deployments; signature update latency can lag
  commercial offerings by hours. Commercial engines and cloud APIs
  have higher detection rates and faster signature updates; cost
  scales with volume.
- *Action on detection*: outright block is the strongest signal to the
  uploader; quarantine is gentler if false positives are common; flag
  is most permissive and least protective. The right answer depends on
  the deployment's risk tolerance.
- *False positives*: every signature-based scanner has them. Provide an
  appeals path: the uploader (or a designated reviewer) can request
  re-scan with a different engine, request operator review, or override
  with logged justification.
- *Privacy*: cloud-API scanners (VirusTotal, etc.) submit file contents
  to a third party. Document this in the deployment's privacy notice;
  for sensitive deployments, on-premises scanning is required.

### Common patterns

| System | Scanning approach |
|---|---|
| Email gateways | Upload-time synchronous with ClamAV or commercial; reject on detection |
| Slack | Upload-time scanning; quarantine-on-detection with admin notification |
| Microsoft 365 (Defender for Storage) | Upload-time + background re-scan; detection triggers admin alerts |
| Most cloud-storage services | Upload-time async with progressive scan-status indicators |
| Internal file servers | Often: no scanning, with reliance on endpoint protection |

### Recommended starting point

For most deployments: **upload-time synchronous scanning with ClamAV**
(or equivalent) **plus weekly background re-scan**. On detection, the
file is rejected at upload time (clear feedback to the user) or
quarantined if detection happens during background re-scan (file is
made undownloadable; uploader is notified; operator is alerted).

Maintain a scan-status field on the FileNode (deployment-defined; not
part of the wire protocol — exposed through implementer-defined
extension or out-of-band signal). States: `clean` / `pending` /
`quarantined` / `failed-to-scan`. Clients SHOULD render the pending
state as "scanning in progress" and the quarantined state as
"unavailable, contact admin".

False-positive appeals: deployments SHOULD provide an admin-mediated
appeals path. The appeal is reviewed by a human (or by a second-opinion
engine); if cleared, the file's status is updated and the user is
notified.

Document the scanning policy in the deployment's privacy notice
(especially if cloud-API scanners are used) and in the user-facing
file-upload UX (so users understand the scan-status indicators).

---

## 4. Quota management

### What the spec leaves open

The filenode draft defines `maxSpaceStorageBytes` (account-level
capability field; max storage across a single Space's file tree) and
references the `overQuota` SetError. It does not specify how to choose
quota values, how to signal soft vs hard limits, how to handle per-account
or per-deployment quotas, or how storage cost is accounted for billing
or reporting.

### What you must decide

- Quota levels: per-Space, per-account, per-deployment, or
  combinations.
- Soft limit (warning) versus hard limit (refuse upload) policy.
- How quotas are configured (per-deployment fixed, per-Space
  adjustable, per-account tied to plan tier, etc.).
- How clients learn approaching-quota state (the wire protocol's
  `overQuota` triggers only at refusal time; advance warning is
  deployment-defined).
- How storage cost is accounted (per-account billing, per-Space
  billing, per-deployment cost center).
- Reclaim policy: when a file is destroyed, when does the space
  actually free up? Same for Space destruction (grace period; see §6).

### Considerations

- *Per-Space quota* (the filenode draft's `maxSpaceStorageBytes`) bounds
  storage growth on a per-conversation basis. Useful for small teams
  with shared budgets; less useful when one Space is a heavy file-sharing
  context and others are mostly chat.
- *Per-account quota* bounds total storage across all Spaces a user
  belongs to. Useful for personal-pay billing models; not directly
  expressible via the wire protocol (which exposes Space-level quotas
  only); deployment-side enforcement.
- *Per-deployment quota* bounds total storage at the deployment level.
  Useful for capacity planning; an unexpected single user or Space
  cannot consume all available storage.
- *Soft vs hard limits*: soft limits emit warnings (visible in client
  UI as "approaching capacity"); hard limits trigger `overQuota`.
  Deployments commonly set the soft limit at 80% of hard; users get a
  warning window to clean up before refusal.
- *Configuration model*: a fixed per-deployment quota for every Space
  is the simplest; per-Space adjustability lets paying customers buy
  more. Per-account tied to plan tier is the SaaS-product model.
- *Advance warning*: the wire protocol does not have a "usage" or
  "quota-remaining" field at the Space level. Deployments may expose
  current usage via deployment-defined extension fields, via a side-channel
  admin API, or via implicit client behavior (e.g., the client
  measures cumulative upload size and warns above a threshold). Server-side
  tracking of usage is necessary regardless; client-visible exposure
  is policy.
- *Cost accounting*: who pays for the storage? Per-Space accounting
  aligns with the wire protocol but does not map cleanly to per-user
  billing. Per-account aligns with billing but requires aggregating
  across Spaces. Per-deployment is operationally simplest but does not
  scale revenue with usage.
- *Reclaim policy*: hard-deleting a FileNode immediately frees storage
  in the operator's metadata view, but the underlying object-storage
  bytes may persist until lifecycle policy reclaims them. Spaces
  destroyed during grace period (§6) hold storage until the grace
  period expires; track this in usage accounting carefully.

### Common patterns

| System | Quota model |
|---|---|
| Slack (free) | Workspace-wide quota; old files deleted past cap |
| Slack (paid) | Per-workspace storage tied to plan; no auto-deletion |
| Discord | Per-message upload size limit (no per-server quota in classic model) |
| Microsoft Teams | Per-team storage tied to underlying SharePoint quota |
| Self-hosted Mattermost | Deployment-configured limits; per-team optional |

### Recommended starting point

- **Per-Space hard quota** via `maxSpaceStorageBytes`: 10 GB as a
  default starting point. Adjust based on deployment scale and per-Space
  usage patterns.
- **Per-Space soft limit at 80% of hard**: clients render an
  "approaching capacity" indicator; uploads still succeed.
- **Per-deployment quota** as an operational safety net: bound total
  storage at the deployment level (e.g., 10 TB for a 1000-user
  deployment as a starting point); a single runaway Space cannot consume
  all available storage.
- **Per-account quota**: only if the deployment's billing model
  requires it. Otherwise, per-Space quotas plus per-deployment caps are
  sufficient.
- **Hard-limit response**: return `overQuota` SetError per the filenode
  draft when the limit is breached. Client renders an explicit
  "Space file storage is full" error.
- **Reclaim accounting**: track usage at the metadata layer (sum of
  FileNode sizes in the Space); reconcile with actual storage usage
  daily to catch drift. Spaces in grace period (§6) MUST count against
  the originating Space's quota; reclaim is deferred until grace
  expiry.

Document the quota policy and the soft/hard signaling in the
deployment's user-facing documentation. Users encountering an
unexpected `overQuota` error should be able to find guidance on how to
free space.

---

## 5. Thumbnail and preview generation

### What the spec leaves open

The filenode draft defines `thumbnailBlobId` and a client-driven
`Blob/convert` workflow for thumbnail generation, conditional on
`urn:ietf:params:jmap:blob2`. It does not specify which file types are
preview-capable, who generates thumbnails (client-side, server-side, a
dedicated rendering tier), or how thumbnail storage is governed.

### What you must decide

- Which file types support thumbnails / previews (images, video,
  PDFs, office documents, source code, etc.).
- Whether thumbnail generation is client-side, server-side, or both.
- How thumbnail blobs are stored and counted against quotas (do
  thumbnails consume the Space quota, or are they free?).
- Cache strategy for derived previews (regenerate on demand vs store
  permanently).
- Privacy: end-to-end-encrypted deployments cannot generate server-side
  thumbnails without breaking encryption.
- Failure handling: when preview generation fails, what does the
  client show?

### Considerations

- *Image previews* are the easiest case: a thumbnail is a smaller
  image. `Blob/convert` per the filenode draft's workflow handles it.
- *Video previews* require a first-frame extract or scrubbing strip;
  more computationally expensive. Some deployments generate at upload
  time; others on first view.
- *PDF previews* render the first page (or a configurable page). Tools
  exist (poppler, MuPDF) but introduce decoder-vulnerability surface.
- *Office document previews* require either LibreOffice headless
  rendering, a cloud rendering service, or a third-party library.
  Heaviest of the common preview types; many deployments skip this.
- *Source code previews* are typically client-rendered (syntax
  highlighting is a client UX concern, not a server one).
- *Client-side vs server-side generation*: client-side keeps the
  server out of the rendering pipeline (lower server cost; better
  privacy posture for E2EE deployments). Server-side ensures
  consistent rendering across client platforms; pays the cost once on
  upload rather than once per client. The filenode draft's workflow is
  client-driven (`Blob/convert` invoked by the client); server-driven
  alternatives are deployment-defined.
- *Thumbnail storage and quota*: a deployment that generates and stores
  thumbnails server-side multiplies its storage cost. Counting
  thumbnails against the Space quota is honest but user-confusing;
  treating them as free is operator-funded. Most deployments treat
  derived previews as operator-funded (do not count toward quota).
- *Cache strategy*: store thumbnails permanently (one-time generation
  cost) versus on-demand regeneration (re-render each time). Permanent
  storage is the common choice; on-demand is appropriate for rarely-viewed
  files at scale.
- *Privacy in E2EE deployments*: the server does not have plaintext;
  server-side thumbnail generation is impossible. The client must
  generate thumbnails locally and upload the (encrypted) thumbnail
  blob.
- *Failure handling*: a corrupt or unsupported file should not break
  the message-rendering flow. Clients SHOULD render a generic file
  icon when no thumbnail is available; preview-generation failures
  should be silent from the user's perspective (logged operator-side).

### Common patterns

| System | Preview generation |
|---|---|
| Slack | Server-side thumbnails for images, video, PDFs, common office docs |
| Discord | Server-side thumbnails for images and short video clips |
| Signal | Client-side thumbnails (end-to-end encrypted; server never sees plaintext) |
| Microsoft Teams | Server-side rendering tier (SharePoint preview generation) |
| Nextcloud | Server-side previews with on-demand generation and caching |

### Recommended starting point

For consumer / social deployments:

- **Image previews**: server-side via `Blob/convert` workflow per the
  filenode draft. Generate at upload time for sizes up to 5 MB; defer
  to client-side or skip for larger images.
- **Video previews**: server-side first-frame extract at upload time
  for videos up to 100 MB; skip for larger videos.
- **PDF previews**: server-side first-page render at upload time;
  bounded by PDF complexity (skip on pages with high render cost).
- **Office documents**: skip in v1. Add later if user demand justifies
  the rendering-tier complexity.
- **Source code**: client-side. The server does not render source.

For E2EE deployments: **all previews are client-side**. The client
generates thumbnails locally before encryption, uploads the encrypted
thumbnail blob, and includes its `blobId` as `thumbnailBlobId`. The
server never sees plaintext content.

Storage and quota: thumbnails are **operator-funded**; do not count
against the Space `maxSpaceStorageBytes` quota. Track thumbnail storage
separately for operator capacity planning.

Cache strategy: **store thumbnails permanently** alongside the source
FileNode. Regenerate only on operator request (e.g., after a thumbnail-format
upgrade across the deployment). Garbage-collect thumbnails when their
source FileNode is destroyed.

Document which file types are preview-capable in the user-facing
documentation. Users should understand which files render inline and
which appear as generic icons.

---

## 6. Retention after Space destruction

### What the spec leaves open

The filenode draft (draft §Space Deletion) establishes that the file tree SHOULD
be destroyed when the Space is destroyed, allows an implementation-defined
grace period during which former members retain access, requires
`forbidden` responses on FileNode operations after grace expiry, and
recommends a state-change event at the moment of destruction. The
specific grace-period duration, the implementation of the grace state
in storage terms, the restoration procedure (if any) during grace,
and the hard-delete-versus-tombstone choice after grace expiry are
deployment-defined.

### What you must decide

- Grace period duration (zero, hours, days, weeks).
- How "during grace period" is represented in storage (soft-delete
  flag on the file tree; access-control list snapshotted at destruction
  time; etc.).
- Whether files can be restored during the grace period (operator
  action; user-initiated request; not at all).
- After grace expiry: hard-delete (remove bytes from storage) or
  tombstone (retain metadata for audit, drop bytes).
- How the deletion interacts with backups (do backups extend the
  effective retention window? does deletion propagate to backups?).
- How compliance and legal-hold requirements layer on top (a Space
  under legal hold may not be deletable on the normal schedule).

### Considerations

- *Zero grace period*: destruction is immediate. Simplest; lowest
  storage cost; offers no recovery from accidental destruction. Risky
  for deployments where Space destruction is a privileged but
  non-reversible operation.
- *Short grace (hours to days)*: handles "oops, didn't mean to" cases.
  Common starting point; balances recovery against indefinite-retention
  cost.
- *Long grace (weeks)*: appropriate for regulated environments where
  destruction-then-restoration cycles are common.
- *Storage representation*: a soft-delete flag on the file tree root
  is the simplest approach. Access-control during grace requires
  snapshotting the membership-and-role state at the moment of
  destruction (so former members retain their access level per the
  filenode draft's Space Deletion rule (draft §Space Deletion)). Production deployments commonly
  use a tombstone record in the metadata DB plus a TTL on the storage
  objects.
- *Restoration*: if restoration is permitted during grace, the
  procedure should be a privileged admin operation, logged to audit
  trail. Restoration re-creates the Space (with a new id, or by
  reviving the destroyed id if not yet purged) and re-activates the
  file tree.
- *Hard-delete vs tombstone after grace*: hard-delete removes both
  bytes and metadata; tombstone retains metadata (FileNode records
  with destroyed status, no associated blob) for audit and legal
  inquiry. Most deployments hard-delete after grace; regulated
  deployments may tombstone permanently or for a long retention period.
- *Backup interaction*: a backup made before destruction contains the
  file tree. If the deployment's backup retention is longer than the
  grace period, the file is recoverable from backup beyond the grace
  window — this is rarely user-facing but matters for
  privacy-compliance ("right to be forgotten" expectations). Document
  the backup-retention-versus-grace-period relationship.
- *Legal hold*: an out-of-band legal-hold flag on a Space MUST
  override the normal destruction schedule. The wire protocol does
  not expose legal-hold state; this is operator-side compliance
  machinery.
- *State-change event*: the filenode draft requires servers to emit a
  FileNode state change at destruction so clients can update without
  polling. Ensure this event fires at grace expiry (the moment the
  file tree actually becomes inaccessible), not at the initial
  Space destruction (which may precede file-tree destruction by the
  grace period).

### Common patterns

| System | Post-deletion retention |
|---|---|
| Slack | Per-workspace setting; default 30 days for deleted channels |
| Discord | Channel deletion is immediate; server deletion has a short cooldown |
| Microsoft Teams | Soft-delete with 30-day default recovery window |
| Google Drive | 30-day trash retention before purge |
| Most file-storage products | 7-30 day soft-delete with admin-controlled retention |

### Recommended starting point

- **Grace period default**: 7 days. Long enough to recover from
  accidental destruction; short enough to bound storage cost.
- **Grace-period representation**: tombstone the Space file tree
  metadata with a destroyed-at timestamp; snapshot the
  membership-and-role state at the moment of destruction; preserve
  access control during the grace window via the snapshot per the
  filenode draft's Space Deletion rule (draft §Space Deletion). Underlying bytes remain in
  storage during the grace period.
- **Restoration**: permitted only by deployment administrators during
  the grace window. Log restoration to audit trail with the actor and
  reason.
- **After grace**: hard-delete metadata and bytes. Tombstone the
  destroyed-Space record (for federation and audit purposes) without
  the file tree contents.
- **State-change events**: emit the FileNode state change at grace
  expiry (the actual destruction moment), not at the initial Space
  destruction event. Clients receiving the change see "the file tree
  is now gone" rather than receiving a misleading event during a
  grace period when the file tree is still technically accessible.
- **Backups**: backup retention SHOULD be coordinated with the grace
  period. A 7-day grace plus 30-day backup retention means files
  remain recoverable from backup for an additional 23 days beyond the
  user-visible grace window. Document this clearly in the
  privacy notice.
- **Legal hold**: an out-of-band flag on the Space (or its parent
  account) MUST suppress destruction-on-schedule. Hold-affected Spaces
  retain their file tree indefinitely until the hold is lifted.

Document the grace period and the restoration procedure in operator
and user-facing documentation. Users destroying a Space should
understand the recovery window; operators should know the formal
restoration steps.

---

## 7. Performance tuning

### What the spec leaves open

The filenode draft and `draft-ietf-jmap-filenode` define the wire
protocol but do not constrain server-side performance characteristics.
Caching strategy, range-request handling, streaming patterns, and
concurrent-upload handling are all operator concerns. The wire
protocol mandates correctness; performance is delivered by the
deployment.

### What you must decide

- Caching strategy: local cache on the application server, CDN
  integration, both.
- Range-request handling for large files (resumable downloads,
  partial fetches).
- Large-file streaming: chunked upload, multipart upload, resumable
  upload via a session token.
- Concurrent-upload handling: throttle limits per account, per Space,
  per deployment.
- Read-after-write consistency: how soon after upload must the file be
  fetchable.
- Geographic distribution: single-region storage, multi-region
  replication, edge caches.

### Considerations

- *Local cache*: the application server caches recently-served blobs.
  Limited capacity; hot files served fast; cold files re-fetched from
  primary storage. Easy to implement; bounded benefit.
- *CDN integration*: blobs served via a CDN edge (Cloudflare, Fastly,
  CloudFront, etc.). High benefit for popular files; cache invalidation
  on destruction or access-control changes adds complexity (a CDN may
  cache a blob after access has been revoked; protect with signed URLs
  or short cache TTLs).
- *Signed URLs* (pre-signed S3 URLs, or equivalent): the application
  server generates a time-limited URL granting blob access; the client
  fetches the blob directly from storage without going through the
  application server. Reduces application-server bandwidth; complicates
  fine-grained access control (the URL is valid for its lifetime
  regardless of subsequent permission changes).
- *Range requests*: large files benefit from resumable downloads
  (browser users on slow networks; mobile users with intermittent
  connectivity). HTTP `Range` header semantics; the storage backend
  must support range responses.
- *Large-file uploads*: a single POST of a 5 GB file is risky (network
  blip kills the upload). Chunked / multipart / resumable upload
  protocols recover from interruption. JMAP's blob upload supports
  chunking; the filenode draft does not constrain this.
- *Concurrent-upload throttling*: a single account uploading 100
  files at once can saturate the application server. Throttle per
  account (e.g., 5 concurrent uploads); apply a queue for additional
  attempts.
- *Read-after-write consistency*: object-storage backends vary.
  Amazon S3 is read-after-write consistent for PUTs since 2020; some
  smaller providers are eventually consistent. The application layer
  should not assume strong consistency unless the chosen backend
  documents it. Common workaround: write metadata only after the blob
  PUT acknowledges; read flow always checks metadata before reading
  the blob.
- *Geographic distribution*: multi-region replication reduces latency
  for distant users but multiplies storage cost. Edge caches via CDN
  give most of the latency benefit without the storage replication
  cost.

### Common patterns

| System | Performance approach |
|---|---|
| Slack | CDN-served file URLs; signed URLs for access control |
| Discord | CDN-served attachments; aggressive edge caching |
| Microsoft Teams (SharePoint backed) | SharePoint CDN; chunked uploads for large files |
| Google Drive | Multi-region replication; edge caches; resumable uploads |
| Self-hosted Nextcloud | Optional CDN; local-server cache; multi-server clustering |

### Recommended starting point

- **Local cache**: the application server caches recently-served blob
  metadata aggressively. Blob bytes are not cached on the application
  server; they are streamed from storage on demand. (Bytes cached at
  the application server level become an invalidation headache; let
  the storage backend or CDN handle byte caching.)
- **CDN integration**: optional in v1; recommended once
  deployment scale justifies the cost. When introduced, use signed
  URLs with short TTLs (5-15 minutes) to bound exposure of revoked
  access.
- **Range requests**: enable for download. Bounded by the storage
  backend's range-request support (S3-compatible: yes; local
  filesystem: yes; some legacy backends: limited).
- **Large-file uploads**: support multipart / chunked uploads via the
  JMAP blob upload mechanism. Resumable upload sessions SHOULD be
  retained for 24 hours after the last activity; abandoned sessions
  cleaned up.
- **Concurrent-upload throttling**: 5 concurrent uploads per account
  as a starting cap; queue additional attempts. Per-Space cap
  optional; per-deployment cap as a safety net.
- **Consistency assumption**: write metadata only after the blob PUT
  acknowledges; read flow always checks metadata before reading the
  blob. This is correct regardless of backend consistency model.
- **Geographic distribution**: single-region in v1. Add multi-region
  replication or CDN edge caches when measured latency or cost
  justifies it.

Document the upload-size limit, the concurrent-upload limit, and the
range-request support in the deployment's user-facing documentation.
Users uploading large files should know what to expect.

---

## 8. Migration patterns

### What the spec leaves open

The filenode draft does not constrain how file trees are migrated
between storage backends, between regions, or between deployments
during merges and reorganizations. Migration is an operator concern,
invisible to the wire protocol when done correctly (the FileNode `id`
and the underlying blob bytes are preserved across the move; clients
see no change).

### What you must decide

- Migration triggers: capacity exhaustion, backend cost shift, regulatory
  data-residency requirements, deployment consolidation.
- Online migration (no user-visible downtime) versus scheduled-downtime
  migration.
- Per-Space migration granularity (move one Space's file tree at a time)
  versus full-deployment migration.
- Cross-region replication during steady state, separate from
  migration events.
- Backup-and-restore procedures (the routine version of migration:
  restore from backup is functionally a migration from backup storage
  to live storage).
- Verification procedure (post-migration integrity check; comparison
  of source and destination).

### Considerations

- *Capacity exhaustion* migration moves data from a full backend to a
  new backend. Typically online: read-only from source, write to
  destination, then cut over.
- *Cost shift* migrations move from expensive storage (often local
  filesystem with redundancy) to cheaper storage (object storage with
  vendor-managed redundancy). Operationally similar to capacity
  exhaustion.
- *Data residency* migrations move data into a specific geographic
  region or vendor for compliance with regulation (GDPR, data
  sovereignty laws). Often time-bounded; failure to complete in time
  has legal consequences.
- *Deployment consolidation* (merging two deployments into one) is the
  most complex migration: ids must be remapped, Spaces and accounts
  reconciled, conflicts resolved. Best done with extended downtime; not
  routine.
- *Online migration* technique: dual-write to source and destination
  for new uploads; background backfill for existing data; cut reads
  over after backfill completes; remove source-side dual-write last.
  Avoid stop-the-world cutover for live deployments.
- *Per-Space granularity*: useful for incremental migrations and for
  multi-region deployments where Spaces have geographic affinity.
  Requires per-Space metadata about which backend or region holds the
  data; routing logic at the application layer.
- *Replication for resilience* (steady state) is operationally similar
  to migration but is continuous and bidirectional. Distinguish:
  replication is for durability; migration is a one-time move.
- *Verification*: post-migration, sample blobs (or check all blobs for
  small deployments) for sha256 match between source and destination.
  Persist verification results; do not consider migration "done" until
  verification is complete and clean.
- *Rollback*: have a documented rollback path until verification is
  clean. The migration is "done" when verification passes; until then,
  the source data is the source of truth.

### Common patterns

- *Dual-write* with background backfill: standard online migration
  technique; well-understood; minimizes downtime.
- *Tar / object-storage rsync* with planned cutover: simpler; requires
  brief downtime during cutover; appropriate for non-critical
  deployments.
- *Backup-and-restore as migration*: restore production backup to a
  new backend; verify; cut over. Appropriate when the backup workflow
  is already proven.
- *Read-through proxy*: route reads through a proxy that knows source
  vs destination; transparent to clients; useful for very-long-running
  migrations.

### Recommended starting point

For most deployments: **dual-write with background backfill** is the
default online migration technique. Concrete procedure:

1. **Pre-flight**: ensure destination backend is provisioned, network
   connectivity is verified, IAM permissions are in place.
2. **Enable dual-write**: every new upload writes to both source and
   destination. New writes are kept consistent; the cost is double
   writes during the migration window.
3. **Background backfill**: walk the source storage; copy each blob
   to destination; verify sha256 match per object. Track per-object
   verification state.
4. **Verification gate**: when 100% of source blobs are backfilled
   and verified, the migration is ready for cutover. Sample-verify a
   subset (e.g., 1%) one more time for confidence.
5. **Cutover**: switch reads to destination. Maintain source-side
   writes for one rollback window (e.g., 7 days) in case rollback is
   needed.
6. **Cleanup**: after rollback window, disable source-side writes;
   remove source-side data per data-retention policy.

Verification: sha256 match per blob is the gold standard; do not skip
this even when both backends report "copy successful".

Per-Space migration granularity: use when migrating to a multi-region
or per-tenant storage layout. Maintain a per-Space backend pointer in
the metadata layer; route reads and writes accordingly.

Backup-and-restore as routine: the deployment's backup procedure
SHOULD be tested as a migration (restore to a clean environment and
verify integrity) at least quarterly. A backup that has never been
restored is not proven to work.

Document the migration procedure and the verification gate in the
operator handbook. Migration is a high-stakes operation; the procedure
should be checklisted and rehearsed.

---

## Cross-references

| Topic | See also |
|---|---|
| Underlying JMAP Chat protocol | `draft-atwood-jmap-chat-00.md` |
| Underlying JMAP File Storage protocol | `draft-ietf-jmap-filenode` |
| The JMAP Chat File Storage spec | `draft-atwood-jmap-chat-filenode-00.md` |
| Permission mapping (Space role → FileNode access) | `draft-atwood-jmap-chat-filenode-00.md` (draft §Permissions) |
| Space-destruction grace-period normative contract | `draft-atwood-jmap-chat-filenode-00.md` (draft §Space Deletion) |
| Controller principal "implicit all permissions" bypass | `jmap-chat-implementer-guide.md` §1 (Governance and roles) |
| Authorization policy for chat content (not file storage) | `jmap-chat-implementer-guide.md` §2 |
| Storage and retention for chat content (not file storage) | `jmap-chat-implementer-guide.md` §4 |
| Federation considerations (file storage is server-local in v1) | `jmap-chat-federation-guide.md` |
| WebSocket transport | `jmap-chat-wss-guide.md` |
| Inline push payloads | `jmap-chat-push-platform-guide.md` |
| Platform-specific push delivery | `jmap-push-platform-guide.md` |
| Calendar integration | `jmap-chat-calendars-guide.md` |
| Task list integration | `jmap-chat-tasks-guide.md` |
