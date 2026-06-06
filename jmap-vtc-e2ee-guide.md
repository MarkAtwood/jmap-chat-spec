# JMAP VTC — End-to-End Encryption Guide

For implementers deploying E2EE with `draft-atwood-jmap-vtc-00`.

Read the spec first, then the E2EE section of the Implementer's Guide
(§4. E2EE deployment). This guide does not restate those decisions. It covers
what real media stacks, SFUs, and gateway protocols can actually do today, and
where the gaps will bite you.

---

## A note on RFC 2119 keywords

This guide uses RFC 2119 keywords (MUST, SHOULD, MAY, REQUIRED, RECOMMENDED,
etc.) for clarity, but in the spirit of implementer guidance rather than as a
normative protocol specification:

- The drafts (`draft-atwood-jmap-vtc-*.md`) are the normative source of truth.
  Where this guide describes a spec requirement using a keyword, the keyword
  reflects the spec's normativity; if guide and draft disagree, the draft wins.
- Where this guide uses a keyword for an operational practice, UX default, or
  deployment choice, the keyword reflects implementer best practice. Deviation
  does not affect protocol interop.
- Cite the spec, not the guide, when claiming normative authority.

---

## 1. The two-layer problem

E2EE in video calling is not one problem — it is two:

1. **Frame encryption.** Each media frame (audio sample, video frame) must be
   encrypted before leaving the sender and decrypted after arriving at the
   receiver. The SFU in the middle forwards ciphertext it cannot read. This is
   the "what gets encrypted" layer.

2. **Key management.** All participants in the call must agree on the current
   encryption key, and that key must change when someone joins or leaves. This
   is the "who has the key" layer.

The spec deliberately stays out of both. The `e2eeEnabled` boolean and
`e2eeFingerprint` string are the only surface: a flag and a verification
handle. Everything else is media-layer-defined. This guide covers what
"media-layer-defined" means when you sit down to build it.

---

## 2. Frame encryption: what works today

### WebRTC Insertable Streams / Encoded Transform

The primary mechanism for WebRTC deployments. The client inserts a transform
between the encoder and the packetizer (sender side) or between the
depacketizer and the decoder (receiver side). The SFU sees encrypted frames
and forwards them by SSRC — it cannot decode, mix, or transcode.

**Browser support (as of 2025):**

| Browser | API | Status |
|---|---|---|
| Chrome / Edge | `RTCRtpScriptTransform` | Stable since Chrome 110 |
| Firefox | `RTCRtpScriptTransform` | Stable since Firefox 117 |
| Safari | `RTCRtpScriptTransform` | Stable since Safari 15.4 |
| React Native (via WebRTC) | Platform-specific | Varies; verify your binding |

The older `createEncodedStreams()` API is deprecated. Use `RTCRtpScriptTransform`
exclusively in new implementations.

**What the SFU must support.** The SFU must forward frames without decoding
them. Not all SFUs do this by default:

| SFU | E2EE forwarding | Notes |
|---|---|---|
| mediasoup | Yes | Forwards opaque payloads; no transcoding path |
| Jitsi Videobridge | Yes | Used in Jitsi Meet E2EE mode |
| Pion (Go) | Yes | Forwards raw RTP; no media processing |
| LiveKit | Yes | E2EE mode documented |
| Janus | Partial | Requires careful plugin configuration; the VideoRoom plugin decodes by default |
| Opal (Opalvoip) | No | MCU architecture; requires decode/re-encode |

If your SFU decodes frames for any reason — simulcast layer selection,
bandwidth estimation via frame inspection, thumbnail generation — E2EE
breaks. Verify that your SFU's simulcast and bandwidth adaptation work on
encrypted payloads (they operate on RTP header extensions, not frame content).

### SFrame (RFC 9605)

SFrame encrypts individual media frames with a symmetric key and a frame
counter. It is transport-agnostic: it works over WebRTC, raw UDP, or
any packetization scheme. SFrame provides a cleaner layering than Insertable
Streams because the encryption is defined independently of the WebRTC API.

**Practical status.** SFrame is standards-track (RFC 9605, published 2024) but
library support is still maturing. The reference implementations are:

- libsframe (C, reference quality)
- SFrame in WebKit (Safari's internal implementation)
- Various Rust/Go ports (check maturity before adopting)

SFrame does not define key management. It takes a symmetric key and a key ID
as input. You need MLS or another key agreement protocol underneath.

**When to prefer SFrame over raw Insertable Streams.** If you need E2EE
across heterogeneous transports (e.g., WebRTC for browsers and a native RTP
path for mobile), SFrame gives you one encryption format for both. If you are
WebRTC-only, Insertable Streams with a custom encryption function is simpler
to deploy today.

### What does not work for E2EE

**SRTP with DTLS-SRTP key exchange.** This is hop-by-hop encryption between
the client and the SFU. The SFU terminates the DTLS handshake and has the SRTP
keys. This is transport security, not end-to-end encryption. Every WebRTC
call already uses DTLS-SRTP — it does not make the call E2EE.

**SRTP with SDES.** Keys are exchanged in SDP, in plaintext over the
signaling channel. The server (and anyone who can read SDP) has the keys.
Not E2EE. SDES is also deprecated by WebRTC.

**ZRTP.** Peer-to-peer key agreement based on a short authentication string
(SAS). Works for point-to-point calls without a server in the media path.
Does not work through an SFU (the SFU would need to participate in the
handshake). Not applicable to group calls.

---

## 3. Key management: MLS is the answer for groups

For a two-party call, a single Diffie-Hellman key agreement suffices. For
group calls — the common VTC case — you need a group key agreement protocol
that handles:

- Adding a participant (they need the current key)
- Removing a participant (remaining participants must rotate to a key the
  removed participant does not have)
- Forward secrecy (compromise of a current key does not reveal past sessions)
- Post-compromise security (a compromised member who is later removed
  cannot read future messages)

**MLS (RFC 9420)** is the IETF standard for this. It is designed for exactly
this use case. It scales logarithmically with group size (unlike pairwise
ratchets which scale quadratically).

### Mapping MLS to VTC participant lifecycle

The key insight: the MLS group membership MUST track the VTCCall's active
participants. Every VTCParticipant join or leave triggers an MLS operation.

| VTC event | MLS operation | Timing |
|---|---|---|
| Participant joins (`joinedAt` set) | MLS Add + Commit | Before media flows |
| Participant leaves (`leftAt` set) | MLS Remove + Commit | Immediately on leave |
| Participant kicked (`kickedBy` set) | MLS Remove + Commit | Immediately on kick |
| Participant reconnects (same userId) | MLS Add + Commit (new leaf) | On reconnect |
| Lobby participant admitted | MLS Add + Commit | On admission, not on lobby entry |
| Call ends | MLS group destroyed | On `state: "ended"` |

**The commit latency problem.** MLS Commit is not instant — it requires a
round of message distribution. During the commit, there is a brief window
where the old key is still in use. For VTC, this means:

- On **join**: the new participant cannot decrypt frames until the Commit
  completes and they receive the new epoch's key. Clients SHOULD show a brief
  "connecting encryption..." state rather than black frames.
- On **leave/kick**: the removed participant can still decrypt frames from the
  old epoch until the Commit completes. This window SHOULD be as short as
  possible (sub-second for well-implemented MLS).

**MLS Delivery Service.** MLS requires a delivery service to distribute
handshake messages (Welcome, Commit, Proposal). The JMAP server can serve
this role — it already knows the group membership. The delivery channel can be:

- The WebSocket connection (as an extension event type)
- The bound Chat (MLS handshake messages as system messages)
- A dedicated out-of-band channel

The JMAP server sees MLS handshake messages but they are cryptographically
opaque — they do not reveal the key material.

**MLS implementations to evaluate:**

| Library | Language | Maturity |
|---|---|---|
| OpenMLS | Rust | Production-ready; used by Wire |
| mls-rs | Rust | Production-ready; used by Wickr/AWS |
| libmls (Cisco) | C++ | Mature; used in WebEx |
| go-mls | Go | Emerging |
| mlspp | C++ | Reference implementation from the RFC authors |

### Point-to-point calls (ring calls, 1:1)

For two-party ring calls, MLS works but is overkill. A simpler approach:

1. Each participant generates an ephemeral X25519 keypair.
2. Exchange public keys via the signaling path (JMAP server relays them —
   it sees the public keys but not the shared secret).
3. Derive a shared secret via X25519 + HKDF.
4. Use the shared secret as the frame encryption key.

The `e2eeFingerprint` in this case is the SHA-256 of the public key.
Participants verify by comparing fingerprints out-of-band.

This avoids MLS complexity for the 1:1 case while remaining compatible — if
a third participant joins (escalating to a group call), transition to MLS at
that point.

---

## 4. Gateway reality

Gateways are the hard boundary for E2EE. Each gateway protocol has different
encryption capabilities, and none of them can participate in an MLS group.

### PSTN

**E2EE capability: none.** PSTN is analog or circuit-switched digital (TDM).
There is no encryption of any kind on the PSTN leg. The gateway transcodes
between the VTC media and G.711/G.729 PCM. To do this, it must decrypt the
VTC media — which means it must have the key — which means the call is not
E2EE for anyone.

**Consequence:** A PSTN participant in a call breaks E2EE for the entire call.
There is no partial E2EE where "everyone except the phone participant" is
encrypted. The gateway is in the key schedule, and it has plaintext audio.

### SIP / SRTP

**E2EE capability: hop-by-hop only.** SIP endpoints can negotiate SRTP via
DTLS-SRTP or SDES. This encrypts the media between the SIP endpoint and the
gateway. The gateway terminates the SRTP and re-encrypts (or doesn't) on the
VTC side. The gateway has plaintext at the transcoding boundary.

**OPAL SRTP (OPAL-VoIP, FreeSWITCH, Opalvoip):** These stacks terminate SRTP
at the gateway. They do not support forwarding encrypted frames into an MLS
group.

**Opal point-to-point exception:** For a direct SIP-to-SIP call (no SFU, no
gateway), SRTP with DTLS-SRTP provides transport encryption between the two
endpoints. This is not E2EE in the JMAP VTC sense (the JMAP server is not in
the media path at all), but it is encrypted. The `e2eeEnabled` flag does not
apply to this scenario — it is a media-layer concern outside the spec.

### H.323 / H.235

**E2EE capability: theoretical, not practical.** H.235 defines encryption for
H.323 media streams, but deployment is rare. Most H.323 endpoints in the
field do not support H.235. Even when they do, the encryption is point-to-point
between the endpoint and the gateway — the same hop-by-hop limitation as SIP.

### Summary: gateway + E2EE decision matrix

| Scenario | E2EE possible? | Action |
|---|---|---|
| All participants are WebRTC | Yes | Full E2EE via Insertable Streams + MLS |
| All participants are native app | Yes | Full E2EE via SFrame + MLS |
| Mix of WebRTC and native | Yes | SFrame for interop, or per-transport encryption with shared MLS key |
| Any PSTN participant | No | Gateway decrypts; entire call loses E2EE |
| Any SIP participant (via gateway) | No | Gateway terminates SRTP; call loses E2EE |
| Any H.323 participant (via gateway) | No | Same as SIP |

**The all-or-nothing rule.** The spec's `e2eeEnabled` is a Boolean on VTCCall,
not per-participant. This is correct: E2EE is a property of the call, not of
individual participants. If any participant cannot participate in the key
agreement, the call is not E2EE.

### What the server SHOULD do

When `e2eeEnabled` is `true` on a call:

- **Reject gateway dial-out.** If a moderator attempts to create a
  VTCParticipant with `joinMethod` of `"pstn"`, `"sip"`, or `"h323"`, the
  server SHOULD return `forbidden` with a description explaining that gateway
  participants cannot join an E2EE call.

- **Reject gateway dial-in.** If a gateway connection arrives for a call with
  `e2eeEnabled: true`, the server SHOULD reject the connection and return an
  appropriate error to the gateway (e.g., SIP location_status location_status location location SIP Location / location SIP 488 Not Acceptable Here).

- **Warn on downgrade.** If a moderator attempts to set `e2eeEnabled: false`
  on an active E2EE call (to admit a gateway participant), this is a security
  downgrade. The server SHOULD require explicit confirmation, and all
  participants MUST be notified (via `VTCRecordingStateEvent` or a new
  event type if warranted).

---

## 5. Feature degradation

When `e2eeEnabled: true`, several features become unavailable because they
require server-side media access. Implementers MUST surface these constraints
clearly in the client UI.

### Degradation matrix

| Feature | Available with E2EE? | Why |
|---|---|---|
| Recording (VTCRecording) | No | Server cannot access plaintext media |
| Livestreaming (VTCLivestream) | No | Server cannot re-encode for RTMP |
| PSTN dial-in/out | No | Gateway cannot join MLS group |
| SIP dial-in/out | No | Gateway terminates SRTP at boundary |
| H.323 dial-in/out | No | Same as SIP |
| Server-side transcription | No | Server cannot access plaintext audio |
| Server-side noise suppression | No | Server cannot process audio frames |
| Simulcast layer selection | Yes* | SFU selects layers by RTP headers, not frame content |
| Bandwidth adaptation | Yes* | SFU adjusts bitrate via REMB/TWCC, not frame content |
| Active speaker detection | Partial | SFU can use audio levels from RTP header extensions (RFC 6464) if client includes them unencrypted |
| Live captions (via Chat) | Client-side only | Client runs local speech-to-text and sends as Chat ephemeral events |
| In-call chat (via Chat) | Yes | Chat messages are independent of media E2EE (Chat has its own E2EE story per {{JMAP-CHAT}} §E2EE) |
| Reactions (via Chat) | Yes | Same as chat |
| Lobby/waiting room | Yes | Lobby is signaling-layer, not media-layer |
| Breakout rooms | Yes | See §6 below |
| Hand raise | Yes | `mediaState.raisedHand` is signaling, not media |

\* Requires SFU that operates on RTP headers without inspecting frame content.

### Client UI obligations

Before a user enables E2EE (or joins an E2EE call), the client SHOULD
display a clear summary of what will be unavailable:

- "Recording and livestreaming are not available in encrypted calls."
- "Phone and SIP participants cannot join encrypted calls."
- "Live captions are device-local only — they are not shared with other
  participants unless your device generates them."

If a moderator attempts a disabled action (e.g., taps "Start recording" on
an E2EE call), the client SHOULD show an explanatory error before the
request reaches the server.

---

## 6. Breakout rooms and key isolation

When a moderator creates breakout rooms from an E2EE parent call, each
breakout room is a separate VTCCall with its own `e2eeEnabled` flag. Key
isolation is critical: a participant moved to Breakout Room A MUST NOT be
able to decrypt media from Breakout Room B or the parent call (unless they
are still in it).

### Implementation approach

Each breakout room gets its own MLS group. When a participant is moved
from the parent call to a breakout room:

1. **Remove** the participant from the parent call's MLS group (Commit).
2. **Add** the participant to the breakout room's MLS group (Welcome + Commit).
3. The participant's `callId` is updated (VTCParticipant/set) to the breakout
   room's VTCCall id.

When the breakout room closes and participants return to the parent call,
reverse the process.

**The moderator case.** A moderator who visits multiple breakout rooms may be
a member of several MLS groups simultaneously. This is fine — MLS supports
a user being in multiple groups. The moderator's client manages multiple
decryption contexts concurrently.

---

## 7. Multi-device

A user may join a call from multiple devices (phone and laptop). The spec
handles reconnection by reusing the existing VTCParticipant record
(§VTCParticipant lifecycle, Reconnection). For E2EE, multi-device raises
a question: is each device a separate MLS leaf, or does the user have one
identity across devices?

### Recommended approach

Each device is a separate MLS leaf node with its own keypair. The MLS group
has one leaf per device, not per user. This matches how MLS is designed
(RFC 9420 §2.1: "Each member of a group is associated with a leaf node").

Consequences:

- Each device has its own `e2eeFingerprint`. The client SHOULD display per-device
  fingerprints when the user expands a participant's details, but show a single
  "verified" badge at the participant level if all of that user's devices are
  verified.
- When a device disconnects, remove its leaf from the MLS group. When it
  reconnects, add a new leaf. Do not reuse old key material.
- The VTCParticipant record is per-user (the spec reuses the record on
  reconnect). The MLS group is per-device. The JMAP layer and the MLS layer
  disagree on cardinality — the implementation must bridge this.

---

## 8. Fingerprint verification

The spec says clients SHOULD display `e2eeFingerprint` for out-of-band
verification (§VTCParticipant Object Fields). Here is what that looks like
in practice.

### Fingerprint format

A full SHA-256 hex dump (64 characters) is too long for anyone to actually
verify. Real systems truncate or re-encode to a length humans will use:

| System | Format | Length | Good for |
|---|---|---|---|
| Signal / WhatsApp | 60 decimal digits (12 groups of 5) | ~199 bits | QR scan; too long to read aloud |
| Wire | 32 hex chars (8 groups of 4) | 128 bits | Visual comparison |
| Keybase | 4-6 English words (PGP word list) | ~44-66 bits | Reading aloud on a call |
| Matrix / Element | 7 emoji blocks | ~42 bits | Quick visual check |

**Recommended default: 4-6 words from a standardized word list.** VTC calls
have a built-in voice channel — the natural verification gesture is "read
your code aloud." Words are dramatically easier to speak and hear than hex
digits or decimal strings. Use a word list with 2048 entries (11 bits per
word); 4 words gives 44 bits of collision resistance, 6 words gives 66 bits.
The BIP-39 English word list or the PGP word list are reasonable choices.

Example (4 words): `"tiger palace running copper"`

For clients that also support visual comparison (showing screens side by
side), display the words alongside a compact hex or numeric encoding as
a secondary format.

**Derivation.** Take the SHA-256 of the participant's public key, truncate
to the desired number of bits, and encode as word-list indices. Store the
full SHA-256 in `e2eeFingerprint` — the display encoding is a client concern.
The `e2eeFingerprint` field value SHOULD be the full hex-encoded SHA-256 for
interoperability; clients choose how to present it.

### Verification UX patterns

| Pattern | How it works | Trust level |
|---|---|---|
| TOFU (trust-on-first-use) | Client stores fingerprint on first encounter; warns on change | Protects against mass surveillance; vulnerable to first-contact MITM |
| Manual verification | Users compare fingerprints out-of-band (read aloud, show screens) | Strong; requires user effort |
| QR code scan | One user displays QR code containing fingerprint; other scans it | Strong; requires physical proximity |
| Cross-signing (PKI) | Server or CA vouches for user-to-key binding | Strongest; requires key infrastructure |

**Recommended starting point.** Implement TOFU with a clear UI warning when
a participant's fingerprint changes. Add a "Verify" button in the participant
list that displays the full fingerprint and instructions for out-of-band
comparison. This covers most threat models without requiring PKI
infrastructure.

### When fingerprints change

A fingerprint change is expected on:

- First join to any call (new ephemeral keypair)
- Device change (new device, new keypair)
- App reinstall (key material lost)
- MLS epoch change (if fingerprint is derived from MLS credential rather than
  raw public key)

A fingerprint change is suspicious if:

- The user did not change devices or reinstall
- The change happens mid-call

Clients SHOULD distinguish these cases in the UI. A "new device" explanation
is reassuring; a mid-call change with no explanation is alarming.

---

## 9. Testing E2EE

E2EE is uniquely difficult to test because success is invisible (encrypted
frames look like random bytes) and failure is silent (the call may appear to
work while sending plaintext).

### What to verify

1. **The SFU cannot read frames.** Capture RTP packets at the SFU. Verify
   that the payload is not valid VP8/VP9/H.264/Opus. If you can decode the
   payload with a standard decoder, E2EE is not working.

2. **Key rotation on leave.** Have participant A join, then leave. Capture
   frames before and after A's departure. Verify that A's recorded key
   cannot decrypt frames sent after A left (the MLS epoch advanced).

3. **Key rotation on join.** Have participant B join mid-call. Verify that B
   cannot decrypt frames sent before B joined (B was not in the MLS group
   during the prior epoch).

4. **Fingerprint accuracy.** Verify that the `e2eeFingerprint` value returned
   by VTCParticipant/get matches the actual public key used by the client's
   encryption layer. A mismatch means the verification UI is lying.

5. **Gateway rejection.** Attempt to add a PSTN participant to an E2EE call.
   Verify that the server returns `forbidden`.

6. **Recording rejection.** Attempt VTCRecording/set create on an E2EE call.
   Verify `forbidden`.

7. **Downgrade notification.** If your deployment supports disabling E2EE
   mid-call, verify that all participants receive a notification before
   plaintext frames flow.

### Automated testing approach

Use two WebRTC clients in a test harness (Puppeteer/Playwright with
`--use-fake-device-for-media-stream`). One client sends a known test pattern.
The other client decrypts and verifies the pattern matches. A third observer
(connected to the SFU's forwarding port) attempts to decode the same stream
and verifies it gets garbage.

---

## 10. Phased deployment

E2EE adds significant complexity. The Implementer's Guide recommends not
implementing it in the first version unless it is a product requirement.
If you do implement it, consider a phased rollout:

### Phase 1: 1:1 ring calls only

- Ephemeral X25519 key agreement (no MLS needed)
- Insertable Streams with AES-GCM encryption
- SHA-256 fingerprints with TOFU
- `e2eeEnabled: true` disables recording and gateway dial-in/out
- Ship and validate

### Phase 2: Group room calls

- Add MLS for group key management
- MLS group lifecycle tracks VTCParticipant join/leave
- Multi-device support (per-device MLS leaves)
- Same fingerprint and TOFU model

### Phase 3: Scheduled calls and breakout rooms

- MLS group creation deferred until first participant joins
- Breakout room key isolation (separate MLS groups)
- Moderator multi-group membership

### Phase 4: Advanced features

- Cross-signing / PKI for enterprise deployments
- SFrame for cross-transport interoperability
- Client-side recording (encrypted locally, uploaded as blob)

Each phase is independently deployable. The `e2eeEnabled` boolean is the
same in all phases — clients do not need to know which phase the server
implements.

---

## 11. Relationship to JMAP Chat E2EE

When both `urn:ietf:params:jmap:vtc` and `urn:ietf:params:jmap:chat` are
deployed, there are two independent E2EE domains:

1. **Media E2EE** (this guide): encrypted audio/video frames via Insertable
   Streams or SFrame, keyed by MLS or ephemeral DH. Controlled by
   `VTCCall.e2eeEnabled`.

2. **Chat E2EE** ({{JMAP-CHAT}} §E2EE): encrypted message bodies, keyed by
   MLS or similar. The relay carries ciphertext; the server is excluded from
   the key schedule.

These are separate key schedules. A call can have E2EE media but unencrypted
in-call chat, or vice versa. In practice, if you deploy one, you SHOULD deploy
both — users expect "encrypted call" to mean everything is encrypted.

**Shared MLS group?** It is tempting to use a single MLS group for both media
keys and chat keys. This works if the membership is identical (same
participants in the call and the chat). It breaks if the Chat has members who
are not in the call (e.g., a persistent channel with an ongoing call). The
safe default is separate MLS groups for media and chat, deriving independent
keys from each.

---

## 12. What the spec does not cover (and should not)

The spec's minimalism is correct. The following are media-layer decisions
that do not belong in a signaling-state specification:

- **Which cipher suite to use.** AES-128-GCM is the common default for
  frame encryption. AES-256-GCM for higher security requirements. The spec
  does not and should not specify this.

- **Key derivation details.** HKDF labels, salt values, context strings —
  these are defined by MLS (RFC 9420) and SFrame (RFC 9605). The spec
  points to these RFCs; it does not restate them.

- **Certificate format.** X.509, raw public keys, MLS credentials — this
  is deployment-specific. The `e2eeFingerprint` field abstracts over it.

- **Rekeying frequency.** How often to rotate keys beyond the join/leave
  triggers. MLS supports periodic rekey; the interval is a deployment
  choice.

- **Recovery from key compromise.** MLS provides post-compromise security
  via Update proposals. The mechanics are in RFC 9420 §12.

The spec's job is to carry the boolean and the fingerprint. Everything else
is your job.
