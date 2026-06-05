# JMAP VTC — Gateway Integration Guide

For implementers integrating PSTN, SIP, and H.323 gateways with `draft-atwood-jmap-vtc-00`.

Read the spec first. This guide does not re-state normative requirements. It
covers what the spec deliberately leaves open and what gateway implementations
must decide before shipping.

---

## How to read this guide

The VTC spec defines VTCGateway as a protocol-opaque abstraction. It knows
that gateways exist, that they have a protocol, that they may support dial-in
and dial-out, and that protocol-specific signals pass through as opaque
VTCGatewaySignal events. It does not define how PSTN carriers connect, how SIP
trunks authenticate, how H.323 gatekeeper registration works, or how to
negotiate codecs across protocol boundaries. Those decisions belong to the
gateway implementation.

This is not a free pass. An implementation that ignores a deferred decision
will produce broken dial-in flows, silent participants, billing surprises, or
security gaps. Implementations must make each of these decisions explicitly,
document them, and implement them coherently.

Each section below follows the same shape:

1. **What the spec leaves open** — with a field or section citation, so you
   can read the normative text yourself.
2. **What you must decide** — the concrete deployment choice you cannot avoid.
3. **Considerations** — the trade-offs that inform the choice.
4. **Common patterns** — how production telephony systems handle this.
5. **Recommended starting point** — a defensible default. Not normative; you
   may choose otherwise with good reason.

When two sections interact (for example, codec choices affect PSTN audio
quality), cross-references are explicit.

This guide is non-normative. The spec is the source of truth. Where this guide
and the spec disagree, the spec wins.

---

## A note on RFC 2119 keywords

This guide uses RFC 2119 keywords (MUST, SHOULD, MAY, REQUIRED, RECOMMENDED,
etc.) for clarity, but in the spirit of implementer guidance rather than as a
normative protocol specification:

- `draft-atwood-jmap-vtc-00` is the normative source of truth. Where this
  guide describes a spec requirement using a keyword, the keyword reflects the
  spec's normativity; if guide and spec disagree, the spec wins.
- Where this guide uses a keyword for an operational or deployment practice,
  the keyword reflects implementer best-practice. Deviation does not affect
  protocol interop.
- Cite the spec, not this guide, when claiming normative authority.

---

## 1. VTCGateway Advertisement

### What the spec leaves open

The spec defines the VTCGateway structure (see `VTCGateway` in the spec) and
its fields: `protocol`, `displayName`, `supportsDialOut`,
`supportedMediaTypes`, `dialInNumbers`, and `metadata`. It does not define how
many gateways to advertise, when to advertise per-call versus account-global
gateways, or what to put in `metadata` for each protocol.

### What you must decide

1. **Which protocols to advertise.** The spec defines `"pstn"`, `"sip"`, and
   `"h323"`. You may define additional values; clients must ignore unrecognized
   ones. Advertise only the protocols your deployment actually operates.

2. **Global versus per-call gateways.** Gateways are advertised in the
   account-level capability object, not on individual VTCCall objects. Every
   call in the account shares the same gateway advertisement. If your
   deployment has gateways available only for specific call types or specific
   organizational units, you must either model that as separate accounts, gate
   it in your dial-out authorization logic, or accept that clients will see
   gateways they cannot use and receive `forbidden` when they try.

3. **What goes in `metadata`.** The `metadata` field is protocol-specific and
   opaque to JMAP VTC. Clients that understand the protocol may inspect it;
   others ignore it. Populate it with whatever your gateway client SDK needs
   to establish a connection.

### Considerations

The `dialInNumbers` array is what gets rendered in the "Join by phone" UI.
Populate it carefully. Every entry should be a number that actually works and
that routes to the correct conference system. Region labels help international
participants pick a local number.

`supportsDialOut` being `true` means moderators will see a "Call in external
participant" UI. If your PSTN carrier charges per-minute for outbound calls,
ensure your authorization layer enforces who may initiate dial-out before
advertising this capability.

`supportedMediaTypes` on a VTCGateway is a subset of the account-level
`supportedMediaTypes`. A PSTN gateway typically supports only `["audio"]`. A
SIP or H.323 gateway connecting to a video-capable endpoint may support
`["audio", "video"]`. Be conservative: only advertise what the gateway can
reliably deliver.

### Common patterns

Production deployments (Jitsi Meet, Zoom, Cisco Webex) typically advertise one
PSTN gateway per region, each with a handful of dial-in numbers. A corporate
deployment may add a SIP gateway representing an internal PBX. H.323 gateways
are advertised when connecting to legacy enterprise MCUs or government
videoconferencing infrastructure.

### Recommended starting point

Advertise one VTCGateway per protocol family you support. Start with a single
region for PSTN and expand to multi-region as operational need requires. Keep
`metadata` minimal: for SIP, the registrar hostname and transport; for H.323,
the gatekeeper address. Omit fields you do not yet support rather than
advertising capabilities you cannot deliver.

Example VTCGateway advertisement in the account-level capability:

```json
"gateways": [
  {
    "protocol": "pstn",
    "displayName": "US PSTN",
    "supportsDialOut": true,
    "supportedMediaTypes": ["audio"],
    "dialInNumbers": [
      {
        "address": "+14155551234",
        "region": "US West",
        "tollFree": false
      },
      {
        "address": "+18005559876",
        "region": "US (toll-free)",
        "tollFree": true
      }
    ],
    "metadata": {}
  },
  {
    "protocol": "sip",
    "displayName": "Corporate SIP Trunk",
    "supportsDialOut": true,
    "supportedMediaTypes": ["audio", "video"],
    "dialInNumbers": [
      {
        "address": "sip:meet@conferencing.example.com",
        "region": "Corporate",
        "tollFree": false
      }
    ],
    "metadata": {
      "registrar": "sip.example.com",
      "transport": "tls"
    }
  }
]
```

---

## 2. SIP Lifecycle Mapping

### What the spec leaves open

The spec defines `joinMethod: "sip"` on VTCParticipant and the VTCGatewaySignal
pass-through mechanism for SIP-specific verbs (SIP REFER, SIP INFO, hold,
unhold, re-INVITE). It does not define how SIP INVITE maps to participant
creation, how BYE maps to participant departure, or how SDP offer/answer flows
through `gatewayData`. Those mappings are your gateway's job.

### What you must decide

1. **Participant identity: SIP URI or authenticated user.** When a SIP
   endpoint registers with your gateway, you may or may not be able to
   associate the SIP identity with a JMAP userId. If you can, set `userId`
   on the VTCParticipant and the participant appears as an authenticated
   user. If you cannot (an external SIP peer with no JMAP account), set
   `userId` to `null` and populate `displayName` from the SIP From header and
   `gatewayData` with the SIP URI.

2. **When to create the VTCParticipant.** Options:
   - On INVITE receipt (before the call is answered). The participant appears
     as "connecting" in the UI.
   - On ACK (after the call is established). The participant only appears
     once media is flowing.
   The spec sets `joinedAt` when the participant joins. Creating the record
   on INVITE with `joinedAt: null` and setting `joinedAt` on ACK is the most
   accurate model.

3. **SDP handling.** The spec does not carry SDP; `joinUri` points to the
   media endpoint. For a SIP gateway, your gateway conducts the SDP
   offer/answer exchange and stores the negotiated parameters in `gatewayData`.
   Decide what to store: at minimum, the negotiated codec and the media
   transport address. Downstream clients that perform SIP-aware reporting can
   then read `gatewayData`.

4. **SIP REFER (call transfer).** REFER triggers a call transfer. Model it as
   a VTCGatewaySignal with `signal: "refer"` and a `data` object carrying the
   Refer-To URI. Moderators send outbound REFER by patching `gatewayData` with
   a `pendingSignal` key (see Section 5 for the signal pattern). The gateway
   extracts it, sends the REFER, and removes the key.

5. **Hold/resume.** SIP hold is a re-INVITE with `sendonly` or `inactive` SDP.
   Map this to a VTCGatewaySignal with `signal: "hold"` or `signal: "unhold"`.
   Optionally mirror the held state in the participant's `mediaState.audio:
   false` so the in-call UI shows the participant as muted.

6. **Registration versus direct dial.** For dial-out, you need to know whether
   the target SIP URI requires registration or can be dialed directly. For
   dial-in, you need a SIP URI that external systems can call. Advertise the
   dial-in URI in `dialInNumbers`.

### Considerations

SIP is stateful on the gateway side but stateless on the JMAP side. Your
gateway maintains the SIP dialog state; JMAP VTC maintains the call and
participant state. Keep these in sync: every dialog state change (INVITE,
ACK, BYE, re-INVITE, CANCEL) must trigger the appropriate VTCParticipant
create or update.

Early media (183 Session Progress with SDP) is common in PSTN-interconnected
SIP flows. Decide whether to represent the early-media phase as a participant
with `joinedAt: null` or to suppress the participant record until the call is
fully established. Showing the participant as "connecting" is better UX for
dial-out flows.

SIP trunking with TLS and SRTP is strongly recommended. See Section 8 for
security considerations. Advertise `"transport": "tls"` in `metadata` to
signal this to clients.

### Common patterns

Jitsi Meet's SIP gateway (Jigasi) creates a conference participant for each SIP
leg when the INVITE is answered. It stores the SIP call ID and negotiated codec
in the participant's metadata. DTMF is relayed as INFO messages or RFC 2833
RTP events; hold/resume maps to re-INVITE SDP changes.

Cisco Webex represents each SIP participant with a SIP URI identity and shows
a distinct "phone" icon in the participant list to distinguish gateway
participants from WebRTC users.

### Recommended starting point

Map the SIP dialog lifecycle to VTCParticipant as follows:

| SIP event          | VTCParticipant action                            |
|--------------------|--------------------------------------------------|
| INVITE received    | create with `joinedAt: null`, `joinMethod: "sip"` |
| ACK received       | update: set `joinedAt` to current time           |
| re-INVITE (hold)   | emit VTCGatewaySignal `signal: "hold"`           |
| re-INVITE (resume) | emit VTCGatewaySignal `signal: "unhold"`         |
| REFER received     | emit VTCGatewaySignal `signal: "refer"`          |
| BYE received       | update: set `leftAt` to current time             |
| CANCEL received    | update: set `leftAt`, `callResponse: "declined"` |

Populate `gatewayData` on the VTCParticipant with the SIP URI and negotiated
codec at a minimum:

```json
{
  "sipUri": "sip:alice@example.com",
  "userAgent": "Linphone/5.2",
  "negotiatedCodec": "OPUS/48000/2",
  "sipCallId": "a84b4c76e66710@pc33.atlanta.com"
}
```

---

## 3. PSTN Integration

### What the spec leaves open

The spec defines `joinMethod: "pstn"` and `gatewayPin` (the PIN an inbound
caller enters to join a specific call). It defines that PSTN participants have
`userId: null` and that `displayName` is derived from caller ID. The
mechanics of how your PSTN carrier connects to your gateway, how PIN
verification works, and how to handle DTMF relay are not prescribed.

### What you must decide

1. **PIN strategy.** `gatewayPin` on VTCCall is the PIN a caller enters to
   join that specific call. Decide whether:
   - Every call gets a unique PIN (most secure; harder to share out-of-band).
   - Room calls have a stable PIN matching the room (more convenient for
     recurring meetings).
   - Scheduled calls use a per-meeting PIN valid only during the meeting window.

2. **Dial-in flow.** The typical PSTN dial-in flow is:
   a. Caller dials the advertised `dialInNumbers` address.
   b. IVR prompts for a conference PIN.
   c. Gateway matches the PIN to a VTCCall via `gatewayPin`.
   d. Gateway creates a VTCParticipant with `joinMethod: "pstn"`, `userId: null`,
      `displayName` derived from caller ID (or "Unknown Caller").
   e. Gateway bridges the RTP stream into the conference mix.
   Implement step (c) as a lookup against active VTCCalls by `gatewayPin`.
   Reject calls with expired or unknown PINs.

3. **Dial-out flow.** A moderator initiates dial-out via `VTCParticipant/set`
   create with `joinMethod: "pstn"` and `gatewayData: {"dialNumber":
   "+14155559876"}`. Your gateway receives this via the VTC server's call to
   your gateway API, dials the number, and when the call is answered, sets
   `joinedAt` on the participant. If the call is not answered, sets `leftAt`
   and records the failure reason in `gatewayData`.

4. **Caller ID mapping.** The PSTN caller ID (CLID/ANI) is unreliable and
   spoofable. Your gateway should:
   - Store the raw ANI in `gatewayData.callerIdNumber`.
   - Optionally store the CNAM (caller name) in `gatewayData.callerIdName`.
   - Not attempt to resolve ANI to a JMAP userId unless you have a verified
     directory mapping.

5. **DTMF relay.** After joining, callers often use DTMF to raise their hand,
   mute themselves, or interact with an IVR menu. Emit each DTMF event as a
   VTCGatewaySignal (see Section 5). Decide which DTMF sequences to
   auto-handle in the gateway (e.g., `*6` to mute/unmute) versus which to pass
   through as signals for the JMAP application layer to handle.

6. **E.164 normalization.** Normalize all phone numbers to E.164 format
   (`+` followed by country code and subscriber number, no spaces or
   punctuation) in `dialInNumbers.address` and `gatewayData.callerIdNumber`.
   This is the only format that is unambiguous across national dialing plans.

### Considerations

PSTN billing is per-minute in both directions. Dial-out calls that ring but
are not answered still incur carrier charges in many jurisdictions. Implement
a ring timeout for outbound PSTN calls (30–60 seconds is typical) and set
`leftAt` on the VTCParticipant when the timeout fires. Log every dial-out
attempt with timestamp, target number, and outcome for billing reconciliation.

PIN brute-force is a real attack vector on PSTN conferencing systems. Impose
a lockout after a small number of failed PIN attempts (3–5) per source number
or per time window. Rotate PINs when a call ends to prevent late joiners from
accidentally entering a past call.

PSTN media is G.711 (alaw or ulaw) almost universally in North America and
Europe. Your gateway will transcode to whatever codec your conference mixer
uses. See Section 7 for codec considerations.

### Common patterns

Zoom generates a unique 9–11 digit meeting ID that doubles as the dial-in PIN.
The dial-in number is regional; the PIN is global across all Zoom numbers.

Google Meet generates a separate 10-digit PIN for phone access to each meeting.
The PIN is valid only while the meeting is active.

Jitsi Meet (with Jigasi) uses the Jitsi conference room name or a numeric alias
as the dial-in code, configurable per deployment.

### Recommended starting point

Assign a random 8–10 digit numeric PIN to each VTCCall at creation time and
store it in `gatewayPin`. Gate PIN length to avoid ambiguity with common
emergency and service numbers. Rotate the PIN when the call ends. Enforce a
5-attempt lockout per source ANI with a 5-minute cooldown. Normalize all
phone numbers to E.164 on ingress.

Example VTCParticipant for a PSTN dial-in caller:

```json
{
  "id": "participant_01HXYZ",
  "callId": "call_01HABC",
  "userId": null,
  "displayName": "Smith, Alice",
  "role": "participant",
  "joinMethod": "pstn",
  "joinedAt": "2026-06-05T14:32:10Z",
  "leftAt": null,
  "gatewayData": {
    "callerIdNumber": "+14155559876",
    "callerIdName": "Smith, Alice",
    "dnis": "+14155551234"
  },
  "mediaState": {
    "audio": true,
    "video": false,
    "screen": false,
    "raisedHand": false
  }
}
```

---

## 4. VTCGatewaySignal Patterns

### What the spec leaves open

VTCGatewaySignal is deliberately opaque. The spec defines the envelope
(`callId`, `participantId`, `protocol`, `signal`, `data`, `direction`,
`timestamp`) and the pass-through mechanism, but does not define what `signal`
strings are valid, what `data` shapes they carry, or which signals are
inbound-only versus bidirectional. Deployments define all of that.

### What you must decide

1. **Signal vocabulary per protocol.** Define the set of `signal` strings your
   gateway emits and accepts for each protocol. Document them. Any client or
   gateway that does not recognize a signal MUST ignore it; the spec requires
   this. But your own clients and gateways need an agreed vocabulary.

2. **Inbound versus outbound.** `direction: "inbound"` means the signal
   originated from the external protocol (the PSTN caller pressed a key; the
   SIP endpoint sent a REFER). `direction: "outbound"` means a moderator
   sent a signal toward the external party. Moderators send outbound signals
   by patching the VTCParticipant's `gatewayData` with a `pendingSignal` key;
   the server extracts it, delivers it to the gateway, and removes the key.

3. **Which signals trigger gateway actions versus JMAP state updates.** Some
   signals are informational (DTMF digit logged for the record). Some require
   gateway action (REFER triggers the gateway to transfer the call). Some
   require JMAP state updates (hold maps to `mediaState.audio: false`). Decide
   the handling for each signal in your vocabulary.

4. **Signal delivery to non-gateway participants.** VTCGatewaySignal events are
   ephemeral WebSocket events. All participants subscribed to VTC events on the
   call will receive them. If your signal vocabulary contains
   protocol-sensitive data (SIP URIs, raw DTMF digits), decide whether to
   suppress delivery to non-moderator participants or to accept that all
   participants can see gateway activity.

### Considerations

The `pendingSignal` mechanism for outbound signals (patching it into
`gatewayData`) is a convention described in the spec. It works for
low-frequency control operations (send a DTMF, initiate a transfer). It is not
a high-throughput channel. Do not attempt to stream audio metadata or frequent
codec-renegotiation events through this mechanism.

DTMF timing matters. RFC 2833 / RFC 4733 carries DTMF as RTP events with
duration. If your application layer needs duration (e.g., for IVR
disambiguation between short and long presses), include `duration` in the
signal `data`. If it does not, omit it to keep the signal compact.

### Common patterns

**DTMF (inbound):**
```json
{
  "@type": "VTCGatewaySignal",
  "callId": "call_01HABC",
  "participantId": "participant_01HXYZ",
  "protocol": "pstn",
  "signal": "dtmf",
  "data": { "digit": "5", "duration": 160 },
  "direction": "inbound",
  "timestamp": "2026-06-05T14:33:05Z"
}
```

**SIP REFER (inbound — external party requesting transfer):**
```json
{
  "@type": "VTCGatewaySignal",
  "callId": "call_01HABC",
  "participantId": "participant_01HXYZ",
  "protocol": "sip",
  "signal": "refer",
  "data": {
    "referTo": "sip:bob@example.com",
    "referredBy": "sip:alice@example.com"
  },
  "direction": "inbound",
  "timestamp": "2026-06-05T14:34:00Z"
}
```

**SIP REFER (outbound — moderator initiating transfer):**

The moderator patches the VTCParticipant:
```json
{
  "update": {
    "participant_01HXYZ": {
      "gatewayData/pendingSignal": {
        "signal": "refer",
        "data": { "referTo": "sip:carol@example.com" }
      }
    }
  }
}
```

The server delivers this to the gateway as an outbound VTCGatewaySignal and
clears `pendingSignal` from `gatewayData`.

**SIP hold (inbound — external party put the call on hold):**
```json
{
  "@type": "VTCGatewaySignal",
  "callId": "call_01HABC",
  "participantId": "participant_01HXYZ",
  "protocol": "sip",
  "signal": "hold",
  "data": { "direction": "sendonly" },
  "direction": "inbound",
  "timestamp": "2026-06-05T14:35:00Z"
}
```

When this signal is received, the gateway SHOULD also update the participant's
`mediaState.audio` to `false` so the in-call roster shows them as muted.

### Recommended starting point

Define your signal vocabulary as a small closed set initially. For PSTN:
`"dtmf"` and `"flash"`. For SIP: `"hold"`, `"unhold"`, `"refer"`, `"info"`,
`"reinvite"`. For H.323: `"h245cmd"` and `"facilityIndication"`. Expand as
specific application requirements emerge. Document the `data` schema for each
signal string. Do not pass raw SIP messages as signal data; extract the
relevant fields.

---

## 5. H.323 / ITU-T Integration

### What the spec leaves open

The spec lists `"h323"` as a valid gateway protocol and provides an H.323
example in `gatewayData` (`{"alias": "alice", "endpointType": "terminal"}`).
It does not define how H.225 call setup maps to VTCParticipant lifecycle, how
H.245 capability exchange flows through the gateway, or how H.235 security
integrates. H.323 is treated identically to SIP and PSTN at the JMAP layer.

### What you must decide

1. **Gatekeeper registration.** H.323 endpoints typically register with a
   gatekeeper (analogous to a SIP registrar). Your gateway must either act as
   an H.323 gatekeeper or register with one. Store the gatekeeper address in
   `metadata.gatekeeper`. The gateway manages the H.225 RAS (Registration,
   Admission, Status) exchange; JMAP VTC sees none of this.

2. **Call setup mapping.** H.225 call setup (Setup → Call Proceeding →
   Alerting → Connect) maps analogously to SIP INVITE flow. Map it to
   VTCParticipant as described in Section 2 for SIP:
   - H.225 Setup received: create VTCParticipant with `joinedAt: null`.
   - H.225 Connect received: set `joinedAt`.
   - H.225 Release Complete: set `leftAt`.

3. **H.245 capability exchange.** H.245 is the H.323 control channel for
   media capability negotiation (analogous to SDP). Your gateway conducts the
   H.245 exchange and handles codec selection. If a capability mismatch
   prevents media flow, reflect this in `gatewayData` (e.g.,
   `"h245Status": "failed"`) and in `mediaState.audio: false`.

4. **H.235 security.** H.235 provides authentication and encryption for H.323.
   H.235 Annex D provides Diffie-Hellman-based key exchange for media
   encryption. Deployments connecting to government or enterprise H.323
   infrastructure often require H.235. Your gateway is responsible for the
   H.235 handshake; JMAP VTC is not involved.

5. **H.323 alias formats.** H.323 endpoints are identified by aliases: E.164
   numbers, H.323 IDs (UTF-8 strings), URL IDs, or transport addresses. Store
   the alias used to identify the endpoint in `gatewayData.alias` and the
   endpoint type in `gatewayData.endpointType` (`"terminal"`, `"gateway"`,
   `"mcu"`, or `"gatekeeper"`).

### Considerations

H.323 is less common in new deployments but remains critical for enterprise
videoconferencing (Cisco, Polycom/Poly legacy hardware) and government
videoconferencing (NATO, US federal systems). If you need to interoperate with
these environments, you need an H.323 gateway.

The H.323 stack is significantly more complex than SIP. Rather than
implementing H.323 from scratch, use an existing library (OpenH323, opal, or
Cisco's H.323 SDK) and write a thin adapter that maps H.323 call events to
VTCParticipant lifecycle calls and VTCGatewaySignal emissions.

H.323 MCUs (Multipoint Control Units) handle multi-party conferencing natively.
When connecting to an H.323 MCU, your gateway may be a single H.323 endpoint
that bridges the JMAP conference to the MCU conference. In this case, the MCU
represents multiple remote participants as a single H.323 leg, which maps to a
single VTCParticipant. This limits per-participant state visibility.

### Common patterns

Jitsi Meet's Jigasi gateway supports H.323 via the Opal library. It represents
each H.323 participant as a single conference participant, with the H.323
alias as the display name.

Cisco Unified Communications Manager (CUCM) acts as an H.323 gatekeeper for
enterprise endpoints. Your gateway registers with CUCM and receives calls
through it.

### Recommended starting point

Unless you have a specific requirement for H.323, start with PSTN and SIP.
Implement H.323 support only when you have a concrete enterprise or government
use case that requires it. When you do implement it, use an existing H.323
library rather than implementing the protocol stack yourself.

Example VTCParticipant for an H.323 endpoint:

```json
{
  "id": "participant_01HXYZ",
  "callId": "call_01HABC",
  "userId": null,
  "displayName": "Boardroom H.323",
  "role": "participant",
  "joinMethod": "h323",
  "joinedAt": "2026-06-05T14:32:10Z",
  "leftAt": null,
  "gatewayData": {
    "alias": "boardroom@videoconf.example.gov",
    "endpointType": "terminal",
    "gatekeeperAddress": "h323gk.example.gov",
    "h245Status": "connected",
    "negotiatedVideoCodec": "H.264",
    "negotiatedAudioCodec": "G.722.1"
  },
  "mediaState": {
    "audio": true,
    "video": true,
    "screen": false,
    "raisedHand": false
  }
}
```

---

## 6. Codec and Media Negotiation

### What the spec leaves open

The spec is deliberately media-agnostic. VTCMediaState tracks whether audio,
video, and screen share are active but not what codec is in use or at what
bitrate. Gateway participants join via `joinMethod` values other than
`"webrtc"`, and `gatewayData` is available for protocol-specific codec state.
The spec does not define how codec negotiation works or what transcoding
decisions to make.

### What you must decide

1. **Where codec negotiation happens.** For WebRTC participants, codec
   negotiation happens in the SDP offer/answer exchange, which is entirely
   outside JMAP VTC. For SIP participants, it happens in the SDP carried in
   INVITE and 200 OK. For H.323 participants, it happens in H.245 capability
   exchange. Your gateway is responsible for all of this. JMAP VTC carries
   only the outcome (which codec was negotiated) as optional metadata in
   `gatewayData`.

2. **What to store in `gatewayData` for codec state.** Consider storing:
   - `negotiatedAudioCodec`: the codec name and parameters (e.g., `"OPUS/48000/2"`,
     `"G.711u/8000"`, `"G.722/16000"`).
   - `negotiatedVideoCodec`: for video-capable gateways (e.g., `"H.264"`,
     `"VP8"`, `"VP9"`).
   - `bandwidth`: negotiated bitrate in kbps, if available.
   This is optional metadata for diagnostic and monitoring purposes; it does
   not affect JMAP protocol operation.

3. **Transcoding strategy.** Your conference mixer operates at a fixed internal
   codec (typically Opus for audio, H.264 or VP8 for video). Gateway
   participants speaking a different codec require transcoding. Decide:
   - **Gateway-side transcoding:** the gateway transcodes before bridging into
     the conference. Simpler architecture; higher CPU cost per gateway leg.
   - **Mixer-side transcoding:** the conference mixer accepts multiple codecs
     natively and handles per-leg decoding. More complex mixer; potentially
     better quality.

4. **PSTN codec constraints.** PSTN carries G.711 (ulaw in North America,
   alaw in Europe) at 8 kHz, 64 kbps. This is narrowband audio. Your gateway
   must transcode G.711 to your internal codec. If your mixer uses Opus, the
   transcoding path is G.711 → PCM → Opus. There is no avoiding this for PSTN
   participants; the PSTN imposes a hard ceiling on audio quality.

5. **Video for SIP and H.323.** Many SIP and H.323 endpoints are audio-only or
   use legacy video codecs (H.261, H.263). Your gateway must handle
   video-capable SIP/H.323 endpoints advertising codecs your conference mixer
   does not support natively. Transcoding H.264 to VP8 or VP9 is
   computationally expensive; consider whether your deployment needs to support
   it.

6. **Bandwidth adaptation.** PSTN has fixed bandwidth. SIP and H.323 endpoints
   may have variable bandwidth. Your gateway should handle bandwidth adaptation
   (bitrate reduction, resolution changes) without propagating those changes
   to JMAP VTC state, since VTCMediaState only carries boolean active/inactive
   flags, not bandwidth levels.

### Considerations

Transcoding adds latency. G.711 to Opus transcoding adds roughly 20–40 ms of
algorithmic delay on top of network latency. For conference calls with mixed
PSTN and WebRTC participants, the total latency budget for PSTN legs is
typically 150–250 ms end-to-end. Plan your transcoding pipeline accordingly.

Comfort noise generation (CNG) and voice activity detection (VAD) matter for
PSTN quality. When a PSTN caller is silent, G.711 carries silence packets. Your
gateway should apply VAD to suppress silence and CNG to generate plausible
background noise on the receiving side, preventing abrupt audio cuts.

### Common patterns

Jitsi Meet uses Opus internally for all audio. PSTN callers (via Jigasi) are
transcoded from G.711 to Opus at the gateway. The conference mixer never sees
G.711.

Zoom uses a proprietary SVC codec internally and handles transcoding for all
external participants (SIP, H.323, PSTN) at the gateway tier.

### Recommended starting point

Use Opus as your internal audio codec. Transcode all gateway audio (G.711 for
PSTN, whatever the SIP/H.323 peer negotiates) to Opus at the gateway. For
video, start with H.264 Baseline Profile as the widest-compatibility choice;
it is supported by essentially all SIP and H.323 video endpoints and all
WebRTC stacks. Store negotiated codec names in `gatewayData` for operational
visibility. Treat video transcoding as a v2 feature unless you have an
immediate requirement.

---

## 7. Security and Identity

### What the spec leaves open

The spec notes that gateway participant identities (`displayName`, `gatewayData`)
are derived from the external protocol's identity mechanisms and are "trivially
spoofable on their respective networks" (see `Gateway Participant Identity` in
the spec). It requires that clients visually distinguish gateway participants
from authenticated JMAP users. It requires that outbound gateway signals be
restricted to moderators. It does not define how to authenticate gateway
connections, how to prevent PSTN caller ID spoofing, or how to prevent billing
abuse.

### What you must decide

1. **Visual distinction in client UI.** The spec requires clients to
   distinguish gateway participants from authenticated users. Define a UI
   convention: a phone icon, a "PSTN" badge, a different avatar style. Apply
   it consistently. Do not rely on `displayName` alone; a spoofed caller ID
   can produce a `displayName` identical to a trusted participant.

2. **Gateway connection authentication.** Your gateway server connects to
   your JMAP server to create and update VTCParticipants and emit
   VTCGatewaySignals. Authenticate this connection with a service account
   token or mTLS client certificate. Never allow unauthenticated writes to
   VTCParticipant records claiming a `joinMethod` of `"pstn"`, `"sip"`, or
   `"h323"`.

3. **PSTN caller ID verification.** PSTN caller ID (ANI/CLID) is spoofable.
   STIR/SHAKEN (RFC 8224 / RFC 8226) provides a signed attestation of calling
   number authenticity, but carrier support is not universal (it is mandated
   in the US and Canada but not globally). Options:
   - Accept caller ID as display-only, with no trust implication.
   - Verify STIR/SHAKEN attestation when available and mark verified callers
     differently in `gatewayData` (e.g., `"stirAttestation": "A"`).
   - Require callers to authenticate via PIN after dialing in (the `gatewayPin`
     mechanism).

4. **SIP authentication.** SIP endpoints that register with your gateway
   should authenticate via SIP Digest authentication or certificate-based
   identity (RFC 4916). External SIP trunks (carrier interconnects) should
   use IP-based allow-listing and/or mutual TLS. Do not accept unauthenticated
   SIP registrations from arbitrary endpoints.

5. **SRTP for gateway media legs.** Media between your conference mixer and
   gateway participants should be encrypted. For SIP, negotiate SRTP via
   SDES (RFC 4568) or DTLS-SRTP. For H.323, use H.235 Annex D. For PSTN,
   media is unencrypted between the PSTN carrier and your gateway; you cannot
   change this, but you can minimize exposure by terminating PSTN media on a
   server with physical access controls.

6. **Billing and abuse prevention for PSTN dial-out.** Unrestricted PSTN
   dial-out is a toll fraud vector. An attacker who gains moderator access to
   a call can initiate dial-out to premium-rate numbers at your expense.
   Controls to consider:
   - Restrict dial-out to a whitelist of allowed country codes or number
     prefixes.
   - Require explicit per-call authorization for international dial-out.
   - Impose per-account and per-day dial-out limits.
   - Alert on anomalous dial-out patterns (many calls in quick succession, calls
     to unusual destinations).

7. **Signal injection.** The spec requires that outbound gateway signals sent
   via the `pendingSignal` mechanism be restricted to moderators. Enforce this
   at the JMAP server level: non-moderators attempting to patch `pendingSignal`
   into `gatewayData` MUST receive `forbidden`. Your gateway must also validate
   that received outbound signals are well-formed for its protocol before
   executing them. Do not execute an arbitrary SIP URI from a `refer` signal
   without validating that the URI is within an acceptable domain.

8. **Recording and E2EE incompatibility.** The spec states that when
   `e2eeEnabled` is `true`, gateway participants are typically unavailable and
   the server SHOULD reject VTCRecording and VTCLivestream creates with
   `forbidden`. This is correct: E2EE media cannot be decoded by a gateway.
   Enforce this in your gateway: if a call transitions to `e2eeEnabled: true`
   while gateway participants are present, evict those participants or notify
   them that they have been disconnected. Do not leave a gateway participant in
   an E2EE call receiving unintelligible media.
   See also *JMAP VTC Implementer's Guide* §4 (E2EE deployment) for the full set of E2EE constraints including recording and livestream restrictions.

### Considerations

PSTN toll fraud is not a theoretical concern. It is a well-documented attack
against conferencing systems that expose PSTN dial-out. The International
Telecommunications Union estimates toll fraud losses in the billions of dollars
annually. The controls in item 6 above are not optional for production systems.

Caller ID spoofing on PSTN is trivially accomplished with readily available
softphone software. Never use caller ID as the primary means of establishing
participant identity. It is display metadata, not an authentication credential.

Gateway participant records with `userId: null` cannot be associated with a
JMAP contact record. This means the blocked-sender check (which the spec
requires before delivering ring notifications) does not apply to gateway
participants. A PSTN caller cannot be blocked via the JMAP contact model;
you need a separate PSTN number block-list mechanism.

### Common patterns

Jitsi Meet shows a telephone handset icon next to PSTN participants. Their
display name is "Phone Participant" or the caller ID name if available.
Moderators can mute or kick them like any other participant.

Zoom requires a meeting PIN for PSTN dial-in and displays "Phone User X" for
unverified callers. Dial-out is restricted to specific plan tiers and geographies.

Enterprise SIP deployments typically use IP allow-listing plus SIP Digest
authentication. Carrier SIP trunks use mutual TLS and RFC 8224 STIR/SHAKEN
where available.

### Recommended starting point

Implement controls in this order, prioritizing the highest-impact risks first:

1. Authenticate gateway-to-JMAP-server connections with a service token before
   anything else. Without this, any network-adjacent attacker can forge
   gateway participant events.
2. Restrict PSTN dial-out by country code whitelist from day one, before
   exposing dial-out to any user. Add international calling as an explicit
   opt-in.
3. Mark all gateway participants with a visual UI badge. Never render them
   identically to authenticated JMAP users.
4. Negotiate SRTP on all SIP gateway legs. Require TLS for SIP signaling.
5. Add STIR/SHAKEN attestation verification as a later iteration when you have
   confirmed carrier support in your target markets.
