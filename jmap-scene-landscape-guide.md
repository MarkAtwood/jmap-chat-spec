# JMAP Scene — Landscape and Prior Art

A technical comparison of JMAP Scene (`draft-atwood-jmap-scene-00`) against the
systems, protocols, platforms, and design traditions that informed it. Covers
what each system got right, what it got wrong or left unaddressed, and how Scene
relates.

This document is non-normative. `draft-atwood-jmap-scene-00` is the source of
truth. Where this guide and the draft disagree, the draft wins.

---

## How to read this document

Scene did not emerge in a vacuum. Every spatial system built in the last forty
years — from Doom to Second Life to Gather to Apple Vision Pro — made design
decisions that Scene either borrows, rejects, or sidesteps. Understanding those
decisions is the fastest way to understand why Scene is shaped the way it is.

Each entry follows the same pattern:

1. **What it is** — a brief factual description.
2. **What it got right** — honest credit for good design decisions.
3. **What is missing or wrong** — limitations, failures, or omissions.
4. **How Scene relates** — what Scene borrows, what it rejects, and why.

The final section synthesizes the patterns across all entries into the design
philosophy that Scene embodies.

This is not a marketing document. Some of these systems are technically
excellent. Some solved problems Scene has not yet attempted. The goal is an
honest accounting, not a sales pitch.

---

## 1. Open Standards and Protocols

### Open Metaverse Interoperability Group (OMI)

**What it is.** OMI is a community group focused on interoperability between
virtual worlds and game engines. Their primary approach is through glTF
extensions (OMI_physics_body, OMI_audio_emitter, OMI_seat, etc.) and behavioral
standards for how objects should work across different engines. They operate
through an open process with public specifications.

**What it got right.** Building on glTF was a sound architectural decision —
glTF is the JPEG of 3D, widely supported and well-specified. The focus on
behavioral interop (not just visual interop) recognizes that a chair is not
just a mesh; it is something you sit on, and that behavior should be portable.
The open process and public specifications are the right governance model.

**What is missing or wrong.** OMI focuses on asset interchange and behavioral
semantics but does not address the session state problem: who is in the room,
where are they, what objects exist right now, who has permission to move them.
You can have perfectly interoperable glTF assets and still have no way to
synchronize a shared spatial environment across clients. OMI provides nouns
(what things are) but not verbs (who is doing what to them, right now).

**How Scene relates.** Scene borrows glTF's coordinate conventions (right-handed,
Y-up, meters, quaternions) and requires `model/gltf-binary` as the
mandatory-to-implement visual format. Scene provides the state layer that OMI
does not address: regions, objects, avatars, permissions, presence. An
OMI-annotated glTF asset is a valid `visualRef` target in Scene. The two
efforts are complementary, not competing.

---

### VRChat OSC Protocol

**What it is.** VRChat exposes avatar parameters (facial expressions, hand
gestures, custom toggles) via Open Sound Control (OSC), a UDP-based protocol
originally designed for music and performance control. External applications
can read and write avatar parameters over OSC, enabling face tracking hardware,
custom controllers, and accessibility tools to drive avatar behavior.

**What it got right.** OSC is simple, real-time, extensible, and well-understood
in the creative technology community. The decision to use an existing protocol
rather than inventing a new one was pragmatic. The parameter model (named floats,
ints, bools) is minimal and sufficient for avatar expression control. It works.

**What is missing or wrong.** OSC has no persistent state — parameters exist
only while the connection is active. There are no spatial queries (you cannot
ask "what is near me"). There is no access control (any OSC client can write
any parameter). There is no schema or type system beyond the four OSC types.
It is a real-time control protocol, not a state management protocol.

**How Scene relates.** Scene's `customProperties` on SceneAvatar serves a
similar role to VRChat's OSC parameters: extensible key-value state attached
to an avatar that the spec does not interpret. The difference is structural:
`customProperties` is persistent JMAP state with standard get/set/changes
semantics, access control, and queryability. Real-time avatar parameter
synchronization (the 10+ Hz updates that OSC handles) is the simulation layer's
job, not JMAP's. Scene provides the state database; the simulation layer
provides the real-time control channel.

---

### IEEE 2888 Series

**What it is.** IEEE 2888 is a family of standards for interfacing between
cyber and physical worlds. IEEE 2888.1 covers sensor data, 2888.2 covers
actuator interfaces, 2888.3 covers digital twin synchronization, and 2888.4
covers virtual-reality disaster response training. The standards define
abstract data models for sensors, actuators, and spatial entities.

**What it got right.** The formal specification process is rigorous. The
sensor/actuator abstraction model is well-thought-out for IoT and digital twin
use cases. The standards correctly identify that cyber-physical interop requires
a common spatial coordinate framework and a common data model for entities.

**What is missing or wrong.** IEEE 2888 is too abstract. The standards define
data models but not wire protocols. There is no practical implementation
ecosystem — no open-source reference implementations, no interop test suites,
no community of implementers building against the spec. The focus is IoT and
industrial digital twins, not social or gaming use cases. The standards read
like academic papers, not implementable specifications.

**How Scene relates.** Scene is more concrete and more opinionated. Where IEEE
2888 defines abstract sensor models, Scene defines specific JMAP data types
with specific methods, specific fields, and specific error conditions. Scene
could serve as the state layer for an IEEE 2888 digital twin scenario — the
SceneRegion is the twin's spatial container, SceneObjects are the twin's
entities — but Scene does not try to model sensors or actuators directly. That
is application logic above the spec.

---

### Croquet OS / Multisynq

**What it is.** Croquet (now marketed as Multisynq) is a distributed computation
framework based on deterministic simulation. Every client runs the same code
with the same inputs and arrives at the same state — bit-identical across all
participants. There is no authoritative server; the "reflector" merely
timestamps and orders messages. The model was invented by Alan Kay's team and
is architecturally elegant.

**What it got right.** The deterministic computation model eliminates the
authority problem: no client is special, no server holds authoritative state,
and consistency is guaranteed by the laws of mathematics rather than by network
protocol. Latency is handled by running the simulation ahead and correcting
when messages arrive. The approach is intellectually beautiful and works well
for applications where all clients can run the same code.

**What is missing or wrong.** Deterministic simulation requires all clients to
run identical code. This means no heterogeneous clients: a browser client and
a native client must produce bit-identical floating-point results, which is
practically impossible across different JavaScript engines, CPU architectures,
and compiler optimizations. It means no thin clients: every participant must
run the full simulation. It means no graceful degradation: a client that cannot
run the simulation cannot participate at all.

**How Scene relates.** Scene deliberately separates state from simulation. The
JMAP server is a state database; it does not simulate anything. The simulation
layer behind `simulationUri` may use Croquet's deterministic model, or a
traditional authoritative server, or client-authoritative peer-to-peer, or no
simulation at all (for static scenes and board games). This separation allows
heterogeneous clients: a full 3D client, a 2D top-down client, a text-only
client, and a mobile client can all participate in the same region, each
running whatever simulation code (or none) is appropriate to their capabilities.
Scene trades Croquet's mathematical elegance for practical heterogeneity.

---

## 2. Corporate Metaverse Platforms

### Meta Horizon Worlds

**What it is.** Horizon Worlds is Meta's consumer virtual world platform.
Meta (formerly Facebook) acquired Oculus in 2014 for approximately $2 billion,
then bet the company on the metaverse pivot — renaming to Meta in 2021 and
pouring an estimated $50 billion or more into the Reality Labs division through
2024. Horizon Worlds launched as Quest-exclusive, later expanded to mobile and
web clients. Users can create worlds using Meta's built-in tools, socialize in
user-created spaces, and attend events. Identity is Facebook/Meta accounts;
there is no federation, no open identity, no interoperability with non-Meta
systems. World state lives on Meta's servers; there is no self-hosting, no
data portability. The protocol is proprietary and unpublished.

**What it got right.** Significant investment in spatial audio — proximity-based
voice in Horizon is genuinely well-implemented. Avatar expressiveness, despite
the initial lack of legs, has improved substantially. The creation tools are
accessible to non-developers. Meta's willingness to invest at scale in spatial
computing, even at enormous financial cost, pushed the industry forward on
hardware quality and manufacturing scale. The Quest headset line made
standalone VR affordable in a way no previous hardware had.

**What it got wrong.** Horizon is a walled garden in every dimension: Meta's
client, Meta's server, Meta's rendering engine, Meta's identity system, Meta's
content moderation, Meta's rules. There is no interoperability with anything
outside Meta's ecosystem. The "metaverse" branding triggered a cultural backlash
that damaged the broader spatial computing industry — people who might have been
interested in virtual worlds became skeptical because of the association with
Meta's corporate ambitions. User engagement numbers have been consistently
disappointing relative to the investment.

The root cause is architectural, not cosmetic. Meta started from the goggles.
Meta acquired Oculus because it controlled a display device, and then built
everything else outward from that hardware relationship. Every design decision
flows from "we own the display" — identity, because the device is tied to a
Facebook account; the server, because the device needs a first-party backend;
the protocol, because there is no reason to publish it when you control the
only client. The result is a platform, not a protocol. This is the AOL
strategy — own the walled garden, extract rent from the network — rather than
the SMTP strategy, which is to define the protocol and let anyone implement it.
AOL had more subscribers than the internet in 1995. It is gone.

This failure mode is distinct from VWRAP's (see Section 3). VWRAP failed by
trying to extract a protocol from an existing implementation — the spec was
too coupled to what Second Life happened to do. Meta is not even attempting
protocol extraction. Meta is building a platform and betting that network
effects will make the walled garden self-sustaining. VWRAP's protocol was too
coupled to one implementation; Meta's platform has no protocol at all. Both
approaches fail for interoperability, but for different reasons: VWRAP was a
protocol that could not escape its implementation; Horizon is an implementation
that was never meant to become a protocol.

**How Scene relates.** Scene is the architectural opposite of Horizon. The
failure-mode mapping is direct:

| Horizon pattern | Scene response |
|---|---|
| Hardware-first (starts from the headset) | Renderer-agnostic (`viewHint` is advisory, not prescriptive) |
| Proprietary identity (Facebook account required) | Opaque `userId` from the auth layer — any identity provider |
| Walled-garden state (Meta servers only) | Standard JMAP data types — self-hostable CRUD |
| No interop (no published protocol) | The protocol is the interop layer; implementations compete on UX |

A Horizon-like experience could be built on Scene + VTC + Chat, but so could
a minimal 2D spatial office, a board game, or a data visualization. Scene does
not require a headset, does not require a Meta account, and does not require
anyone to say the word "metaverse." The difference is not in what experiences
are possible — Horizon Worlds and Scene can render similar experiences. The
difference is in who controls the protocol. In Horizon, Meta controls it. In
Scene, nobody does.

---

### Microsoft Mesh

**What it is.** Microsoft Mesh is an enterprise spatial computing platform
integrated with Microsoft Teams. It evolved from the HoloLens mixed reality
research program. Users can join Teams meetings as avatars in 3D spaces, view
shared content spatially, and collaborate in immersive environments. It targets
enterprise remote collaboration.

**What it got right.** The enterprise use case focus is pragmatically sound —
enterprise customers pay for productivity tools and will tolerate imperfect
technology if it solves a real workflow problem. Teams integration provides a
built-in distribution channel: users do not need to install a new application,
learn a new identity system, or convince their IT department to approve a new
vendor. The avatar system supports both immersive (VR/MR headset) and
non-immersive (desktop) participation in the same meeting.

**What is missing or wrong.** Mesh is proprietary and Azure-dependent. The
protocol is not published; there is no way for a non-Microsoft server to host
a Mesh environment. The platform is limited to the Microsoft ecosystem — you
need Teams, you need Azure, you need Microsoft identity. Cross-vendor spatial
collaboration is not possible.

**How Scene relates.** Scene + VTC + Chat achieves a similar integration model
through open standards. A SceneRegion with an `activeCallId` binding to a
VTCCall and a `chatId` binding to a Chat provides the same three-layer
experience (spatial presence + voice/video + text) that Mesh provides within
Teams. The difference is that Scene's layers are independent JMAP capabilities
that any conforming server can implement, not a proprietary platform tied to
one vendor's cloud.

---

### Apple Vision Pro / visionOS

**What it is.** Apple Vision Pro is Apple's spatial computing headset, running
visionOS. It focuses on "spatial computing" — placing digital content in
physical space via high-quality passthrough — rather than "virtual reality" or
"metaverse." Apple deliberately avoids metaverse branding and positions the
device as a personal computing platform, not a social one.

**What it got right.** Refusing the metaverse label was strategically brilliant.
The focus on practical spatial computing — watching movies, running apps,
productivity — rather than "building a metaverse" avoids the cultural baggage
that damaged Meta's efforts. The passthrough quality is the best in the
industry. The interaction model (eye tracking + hand gestures) is genuinely
innovative. The insistence on high build quality and refusing to ship until the
experience met Apple's bar is the right engineering discipline.

**What is missing or wrong.** visionOS is a completely closed ecosystem. There
is no multi-user spatial protocol — SharePlay exists but is limited to Apple
devices and Apple's frameworks. There is no interoperability with non-Apple
spatial computing systems. Two Vision Pro users cannot share a spatial
experience with a Quest user or a desktop user. The price point ($3,499 at
launch) limits adoption. Apple has shown no interest in open spatial protocols.

**How Scene relates.** Scene could serve as the multi-user state layer that
visionOS does not provide. A visionOS application using Scene would gain
cross-platform multi-user spatial presence — Vision Pro users, Quest users,
desktop users, and mobile users in the same SceneRegion. Scene does not compete
with visionOS's single-user spatial computing; it provides the multi-user
layer that Apple has not built.

---

### Amazon Chime Spatial View

**What it is.** Amazon Chime briefly experimented with a spatial video layout:
a 2D top-down view where participant video tiles were arranged spatially, and
audio was proximity-based — participants closer together in the 2D space heard
each other more clearly. It was an internal experiment that demonstrated the
concept but never shipped as a generally available product.

**What it got right.** The experiment proved that 2D spatial is compelling for
meetings. Proximity-based audio in a 2D top-down layout creates a sense of
presence and enables the "hallway conversation" pattern: small groups can form
organically within a larger meeting space, just as they do at physical
conferences. The experiment validated that you do not need 3D or VR headsets
to get meaningful spatial interaction.

**What is missing or wrong.** It never shipped. There is no public protocol,
no published design document, no open-source implementation. The experiment
demonstrated the concept but did not produce a product or a standard.

**How Scene relates.** Scene's `viewHint: "2d-topdown"` was directly inspired
by this concept and by Gather (see Section 5). A SceneRegion with
`viewHint: "2d-topdown"`, an `activeCallId` binding to a VTCCall, and avatars
with 2D positions implements exactly the spatial meeting layout that Chime
experimented with — but as an open protocol that any client or server can
implement.

---

## 3. Persistent Virtual Worlds

### Second Life

**What it is.** Second Life, launched by Linden Lab in 2003, is a persistent
virtual world with user-created content. It has been in continuous operation
for over 20 years. Users own virtual land, create objects with a built-in
scripting language (LSL), buy and sell virtual goods in a virtual economy
(Linden Dollars, convertible to USD), and socialize. Voice chat was originally
provided by Vivox (a separate voice service) and later supplemented with
WebRTC.

**What it got right.**

- *Persistence.* The world exists whether or not you are logged in. Objects you
  place stay where you put them. This is the defining feature that separates
  virtual worlds from games.
- *User-created content.* Users build the world, not the platform operator. This
  scales content creation beyond what any company can produce internally.
- *Permission model.* Objects have owner, group, and everyone permissions (read,
  modify, copy, transfer). This is simple, well-understood, and sufficient for
  most use cases. Land parcels have separate permissions controlling who can
  enter, who can build, and who can run scripts.
- *Separation of voice from world.* Voice (Vivox) runs on a completely separate
  infrastructure from the world server. This is the right architecture: voice
  is a different kind of real-time data with different latency requirements,
  different scaling characteristics, and different failure modes.
- *Virtual economy.* Linden Dollars created real economic incentives for content
  creation. Creators could earn real money, which bootstrapped a content
  ecosystem.

**What it got wrong.**

- *Proprietary protocol.* The Linden Lab Structured Data (LLSD) protocol was
  never formally specified as an open standard. It was documented, reverse-
  engineered, and eventually opened enough for third-party viewers, but never
  became an IETF RFC or a W3C recommendation. Linden Lab did attempt IETF
  standardization (the VWRAP effort, 2009-2011) but it failed — see the
  dedicated VWRAP section below.
- *Viewer monoculture.* For years, the official Linden Lab viewer was the only
  usable client. Third-party viewers eventually appeared (Firestorm, Kokua)
  but they all implement the same protocol and rendering stack. There is no
  thin client, no mobile client (until recently), no 2D client.
- *Performance.* User-created content with no polygon budgets, no LOD
  requirements, and no optimization constraints means the renderer must handle
  arbitrarily complex geometry. This produces consistently poor frame rates in
  populated areas.
- *Land as bottleneck.* The land model (fixed-size regions on dedicated server
  hardware) creates artificial scarcity and limits scaling. Region crossings
  (moving between server boundaries) remain unreliable after 20 years.

**How Scene relates.** Scene borrows several design elements from Second Life:

- *Region/permission model.* SceneRegion's `accessPolicy` and SceneObject's
  `ownerId` follow SL's pattern of separating region-level and object-level
  permissions.
- *Separation of voice.* Scene (spatial state) and VTC (voice/video) are
  independent capabilities, mirroring SL's separation of world server and
  Vivox.
- *Owner/region-owner/admin tiers.* The SceneObject permission model
  distinguishes between the object owner, the region owner, and server
  administrators — the same three-tier model SL uses.

Scene diverges from Second Life in three critical ways: open protocol (JMAP,
not LLSD), format-agnostic visuals (`visualRef` + `visualType`, not a built-in
mesh format), and no built-in economy (virtual currencies and trading are
application logic above the spec).

---

### OpenSimulator

**What it is.** OpenSimulator (OpenSim) is an open-source server that
implements enough of the Second Life protocol to support SL-compatible viewers.
It has been developed since 2007. The Hypergrid feature allows avatars to
teleport between independent OpenSim grids, providing a form of cross-server
federation.

**What it got right.** OpenSim proved that the SL model could be open-sourced
and that independent server operators could run interoperable grids. Hypergrid
demonstrated cross-server teleportation — a user on grid A could visit grid B
with their avatar and inventory, which is a form of federated spatial presence.
The project showed that an open implementation of a virtual world protocol is
possible and that communities will self-organize around it.

**What it got wrong.** OpenSim was perpetually chasing a moving target: the SL
protocol was never formally specified, so OpenSim implemented against
observed behavior and reverse-engineered packet captures. When Linden Lab
changed the protocol, OpenSim had to reverse-engineer the changes. The protocol
was never formally specified as a standard — it was an implementation, not a
specification. This means there is no compliance test suite, no formal
interoperability guarantee, and no way to implement a conforming server without
studying the OpenSim source code.

**How Scene relates.** Scene provides what OpenSim never had: a formal protocol
specification that multiple independent implementations can target. An
OpenSim-like project built against Scene would implement JMAP methods
(`SceneRegion/get`, `SceneObject/set`, `SceneAvatar/set`) against a published
specification with defined error conditions, defined data types, and defined
semantics. Two independent implementations that pass the same conformance tests
would interoperate. The specification is the contract, not the source code.

---

### VWRAP (Virtual World Region Agent Protocol)

**What it was.** VWRAP was Linden Lab's attempt to standardize the Second
Life protocol as an IETF specification. It went through three naming
iterations — MMOX (Massively Multi-Player Games and Applications, BoF at
IETF 74 in March 2009), OGPX (Open Grid Protocol), and finally VWRAP —
before being chartered as an IETF working group in September 2009 and
formally concluded in May 2011 without producing a single RFC.

The effort was primarily driven by Meadhbh Hamrick and Mark Lentczner
(Linden Lab) with David Levine (IBM). It produced roughly a dozen
Internet-Drafts covering introduction/goals, authentication, an abstract
type system (LLSD/LLIDL), client launch messages, and deployment patterns.
Every draft reached only -00 or -01 revision before expiring.

**What it got right.** The effort demonstrated genuine institutional
willingness from Linden Lab and IBM to open up virtual world protocols.
The separation into authentication, type system, and application protocol
layers was architecturally sound in principle. The use of
capability-based security (capability URLs rather than ambient authority)
was ahead of its time.

**What it got wrong.**

- *Coupling with the implementation.* Crista Lopes (UC Irvine,
  "Diva Canto" in OpenSimulator) delivered the definitive critique: the
  VWRAP drafts were not a generalized virtual world interoperability
  protocol but a wire-format description of what the Second Life viewer
  already did. The LLSD type system, the LLIDL interface description
  language, the capability-URL security model, the event queue mechanism —
  all were direct lifts from SL's existing internal architecture. A
  virtual world with a different architecture (peer-to-peer truth model,
  different asset pipeline, different region model) would have had to
  adopt SL's entire worldview to implement the protocol. It was not a
  specification that happened to have an implementation; it was an
  implementation that was retroactively formatted as a specification.
- *Single sponsor, single implementation.* Virtually all drafts were
  authored by Linden Lab employees. When Linden Lab laid off 30% of its
  staff in June 2010 and suspended its involvement in virtual world
  interoperability, the effort lost its only source of technical momentum.
  The only systems the protocol could apply to were Second Life and
  OpenSimulator (which reverse-engineered SL).
- *Community rejection from day one.* At the first MMOX BoF, participants
  from non-SL virtual worlds (Forterra/IMVU, others) disengaged when it
  became clear the effort was about standardizing OGP specifically, not
  building a protocol any virtual world could use. The attempt to split
  into OGP-specific vs. broader interoperability tracks "touched a nerve"
  and was never resolved.
- *Alien to web developers.* Hamrick herself later noted that the
  technical documentation was "relatively foreign to the experience of
  most web developers," which limited the reviewer and implementer pool.
  The LLSD type system was a custom serialization format (not JSON, not
  XML Schema, not Protocol Buffers) that required learning a new IDL.

**How Scene relates.** VWRAP is the cautionary tale that Scene is
designed to avoid repeating. The failure modes map directly to Scene
design decisions:

| VWRAP Failure | Scene Response |
|---|---|
| Custom type system (LLSD) | JMAP's existing JSON type system (RFC 8620) |
| Custom transport | RFC 8620 HTTP + RFC 8887 WebSocket |
| Custom auth | Standard JMAP authentication |
| Coupled to one viewer | viewHint-driven, any renderer |
| Single implementation | Spec-first, no reference implementation required |
| Single sponsor | Built on IETF-published JMAP foundation with existing implementations |
| Alien to web developers | JSON, HTTP, WebSocket — standard web stack |

The core lesson: a protocol specification must be designed independently
of any particular implementation. VWRAP attempted to standardize what
Second Life *happened to do*. Scene specifies what a spatial state
protocol *should do* using primitives (JMAP methods, JSON data types,
WebSocket events) that have proven interoperability across multiple
independent implementations in the email and calendaring domains.

---

### Decentraland

**What it is.** Decentraland is a browser-based virtual world where land
ownership is tracked on the Ethereum blockchain. Users purchase LAND tokens
(ERC-721 NFTs) that correspond to parcels in a fixed grid. Content on each
parcel is served by the land owner. The world renders in the browser using
Babylon.js.

**What it got right.** Decentralized ownership is a legitimate design goal —
the idea that no single entity controls who can own land or what can be built
on it has philosophical merit. Browser-based rendering (no install required)
lowers the barrier to entry. The fixed coordinate grid provides a shared
spatial frame of reference.

**What it got wrong.** Blockchain adds latency, complexity, and cost without
clear UX benefit for most users. Land scarcity is presented as a feature (it
creates economic value) but is actually a limitation (it caps the size of the
world). The user experience is poor: rendering quality is limited by browser
constraints, content quality varies wildly across parcels, and the world feels
empty because the user base is small relative to the amount of land. The
economic model attracts speculators more than creators or socializers.

**How Scene relates.** Scene is orthogonal to Decentraland's ownership model.
Scene does not prescribe ownership, economy, or governance. A blockchain-based
ownership layer could theoretically sit above Scene — land ownership tracked
on-chain, with the chain controlling who has permission to create SceneRegions
and SceneObjects. Scene provides the spatial state protocol; the ownership
and economic layers are application logic above the spec. Scene's access
control (`accessPolicy`, `ownerId`) provides the enforcement hooks; what
populates those fields is the deployment's business.

---

### Roblox

**What it is.** Roblox is a game creation platform with massive scale —
hundreds of millions of monthly active users, primarily young. Users create
games ("experiences") using Roblox Studio and the Luau scripting language.
The platform handles hosting, scaling, monetization, and social features.
It is cross-platform: PC, mobile, console, and recently VR.

**What it got right.** User-created content at scale is Roblox's defining
achievement. Millions of experiences exist, many created by teenagers with
no formal programming training. The cross-platform story is strong — the
same experience runs on a phone, a PC, and a PlayStation. Social features
(friends, chat, voice) are integrated by default, not bolted on. The
economic model (Robux) gives creators real financial incentives.

**What is missing or wrong.** Roblox is completely proprietary. Luau is a
Roblox-specific language (a Lua derivative). Roblox Studio is the only
creation tool. Roblox servers are the only hosting option. There is no way
to take a Roblox experience and run it elsewhere. There is no interoperability
with any other platform. If Roblox shuts down, every experience built on it
disappears.

**How Scene relates.** Scene provides the spatial state layer; a Roblox-like
creation platform could be built on top of Scene + a scripting layer. The
SceneRegion is the experience. SceneObjects are the game entities. SceneAvatars
are the players. `customProperties` carries game-specific state. The scripting
layer and creation tools are above the spec. Scene does not try to be Roblox;
it tries to be the protocol layer that makes open alternatives to Roblox
possible.

---

## 4. Social VR

### Mozilla Hubs

**What it is.** Mozilla Hubs (2018-2024) was a browser-based social VR
platform. Users joined rooms via URL, saw each other as avatars, communicated
via WebRTC voice with spatial audio, and could share media (images, videos,
3D models) into the room. It was open source (the client, Hubs Cloud for
self-hosting, and the Reticulum server). Mozilla shut it down in 2024 due
to funding constraints. A community fork (Third Room, later others) attempted
to continue the work.

**What it got right.**

- *Browser-first.* No install, no app store, no approval process. Click a link,
  you are in a room. This is the right entry model for casual social
  experiences.
- *WebRTC for voice.* Using the browser's built-in WebRTC stack for voice chat
  meant no additional software, no codec licensing, no custom audio pipeline.
  Spatial audio (distance-based attenuation) made presence feel real.
- *Room-based entry model.* Each room is a URL. You share a link, people join.
  No accounts required for guests. This mirrors how the web works: URLs are
  the universal addressing scheme.
- *Open source.* Hubs Cloud allowed self-hosting. The client was open source.
  Reticulum (the server) was open source. This was the right governance model.

**What it got wrong.** The funding model was unsustainable. Mozilla funded Hubs
as a research project and a loss leader for Firefox ecosystem relevance, not as
a self-sustaining product. When Mozilla's financial constraints tightened, Hubs
was shut down. The technology was sound; the business model was not.

**How Scene relates.** Hubs is the primary architectural inspiration for Scene.

- *SceneRegion / SceneAvatar* mirrors Hubs' room / participant model. A
  SceneRegion is a Hubs room. A SceneAvatar is a Hubs participant.
- *Browser-first philosophy.* JMAP is HTTP-native. A Scene client can be a web
  page that makes JMAP requests over HTTPS. No special protocol, no custom
  binary transport, no native SDK required.
- *Spatial audio concept.* Scene's integration with VTC (spatial audio as a
  simulation-layer concern at the intersection of VTC and Scene) follows Hubs'
  approach.

Scene is the spiritual successor to Hubs in terms of architecture, with one
critical difference: Hubs was a platform (client + server + media stack,
tightly coupled). Scene is a protocol. When Mozilla shut down Hubs, everything
built on it died. When a Scene server shuts down, clients can connect to a
different Scene server. The protocol survives the platform.

---

### Mozilla Social

**What it is.** Mozilla Social (2023-2024) was Mozilla's attempt at a
federated social platform, built on Mastodon. It was a Mastodon instance
with Mozilla branding and a vague plan to add spatial features. Mozilla shut
it down in late 2024, roughly six months after launch.

**What it got right.** Recognizing that federation matters was correct. The
insight that social presence should not be controlled by a single platform
operator is sound. The choice of Mastodon / ActivityPub as the federation
protocol was pragmatic — it had an existing ecosystem and user base.

**What it got wrong.** Mozilla Social pivoted away from spatial before it ever
shipped spatial features. It was a Mastodon instance, not a spatial platform.
The federation model (ActivityPub) is designed for text and media posts, not
for real-time spatial state synchronization. ActivityPub's eventual consistency
model and its focus on content distribution rather than session state make it
a poor fit for spatial presence.

**How Scene relates.** Scene achieves what Mozilla Social aspired to — federated
spatial presence — through JMAP's existing architecture rather than ActivityPub.
JMAP already has account-level capability negotiation, authentication,
authorization, and state synchronization. Scene adds spatial data types to this
existing infrastructure. Federation between JMAP servers is a JMAP-level
concern, not a Scene-level concern; Scene inherits whatever federation model
JMAP provides.

---

### VRChat

**What it is.** VRChat is the dominant social VR platform. Launched in 2017, it
has millions of users and a massive community of avatar and world creators.
Users can upload custom avatars (Unity-based), create worlds (also Unity-based),
and socialize with full-body tracking support. VRChat has the largest and most
active social VR community.

**What it got right.** Avatar expressiveness is VRChat's strongest feature.
Full-body tracking, face tracking (via OSC), custom shaders, and the ability
to upload any Unity avatar means users can express themselves with a richness
no other platform matches. The community-driven content model works: the best
VRChat worlds and avatars are made by users, not by VRChat Inc. The OSC
integration (see Section 1) enables an ecosystem of external tools.

**What is missing or wrong.** VRChat is completely proprietary. There is no
formal protocol specification — the networking layer is Unity's UNET (later
replaced with custom networking) and is not documented. Worlds are Unity
projects compiled to VRChat's format; you cannot run a VRChat world on a
non-VRChat server. There is no interop with any other platform. The Unity
dependency means world creation requires Unity (a 10+ GB install with a
learning curve).

**How Scene relates.** Scene could theoretically serve as an interop layer
between VRChat-like platforms — a common state protocol that different social
VR platforms could use to share presence information. In practice, VRChat has
no incentive to adopt open protocols; its value proposition is its community
and content library, both of which are locked to the platform. Scene's
relationship to VRChat is aspirational: Scene provides the protocol that would
make VRChat-to-other-platform interop possible if the platforms chose to
adopt it.

---

### Rec Room

**What it is.** Rec Room is a cross-platform social gaming platform. Users
create rooms and games using built-in creation tools (Circuits, the Maker Pen).
It runs on PC, mobile, PlayStation, Xbox, Quest, and PSVR2. The platform
emphasizes accessibility — creation tools are usable by non-developers, and
the art style is deliberately simple to run well on mobile hardware.

**What it got right.** Cross-platform reach is Rec Room's strength. The same
room works on a phone, a console, a PC, and a VR headset. This is the right
approach — spatial experiences should not require specific hardware. The
creation tools are accessible to non-developers, which enables a large creator
community. The simple art style is a pragmatic engineering decision: it runs
on mobile, which is where most users are.

**What is missing or wrong.** Rec Room is proprietary. Rooms exist only on Rec
Room's servers. The creation tools produce content in Rec Room's format. There
is no export, no interop, no open protocol. If Rec Room shuts down, everything
built on it disappears.

**How Scene relates.** Rec Room and Scene have similar scope — rooms with
objects and avatars that users interact with — but Scene is a protocol, not a
platform. A Rec Room-like experience could be built on Scene (SceneRegion as
the room, SceneObjects as game entities, SceneAvatars as players, interactions
via `SceneInteractionEvent`), but Scene does not provide creation tools, art
styles, or a game engine. Those are above the spec.

---

### AltspaceVR

**What it is.** AltspaceVR (2015-2023) was an early social VR platform,
acquired by Microsoft in 2017. It hosted events, meetups, and social
gatherings. It was the first social VR platform to gain significant traction.
Microsoft shut it down in March 2023 in favor of Microsoft Mesh.

**What it got right.** AltspaceVR was an early mover that proved social VR
was compelling. The events model (scheduled gatherings in virtual spaces)
worked well and attracted real communities. The platform demonstrated that
social presence in VR creates genuine human connection.

**What it got wrong.** AltspaceVR could not find a sustainable business model.
Microsoft acquired it, invested in it, and then shut it down when Mesh became
the preferred direction. Everything built on AltspaceVR — communities, content,
events, social connections — disappeared when Microsoft flipped the switch.

**How Scene relates.** AltspaceVR is a cautionary tale. Platforms shut down;
protocols persist. HTTP did not shut down when Netscape went bankrupt. SMTP
did not shut down when any particular email provider closed. Scene is a
protocol, not a platform. A Scene server can shut down, but the protocol
continues to work on every other Scene server. The communities and content
built on Scene are portable because the protocol is open and the data types
are standard. This is the single most important architectural lesson from
AltspaceVR's shutdown.

---

## 5. Spatial Office and Collaboration

### Gather

**What it is.** Gather (2020-present) is a browser-based spatial video platform.
Users appear as 2D pixel-art avatars in a top-down 2D map. Audio and video are
proximity-based: you hear and see people near your avatar, with volume
attenuating over distance. The primary use case is virtual offices — teams
leave Gather open all day and move their avatar to different "rooms" (areas on
the map) to have conversations. It launched during the COVID-19 pandemic and
found product-market fit with remote teams.

**What it got right.**

- *2D spatial is compelling.* Gather proved that you do not need 3D, VR, or
  headsets for meaningful spatial interaction. A 2D top-down map with proximity
  audio creates genuine presence and enables spontaneous conversations.
- *Proximity-based interaction is intuitive.* Walking your avatar near someone
  to talk to them is immediately understood by everyone. No tutorial needed.
- *Low barrier to entry.* Browser-based, no install, no hardware beyond a
  laptop. This is the right entry point for workplace tools.
- *Always-on presence.* The virtual office pattern — leave Gather open, your
  avatar is visible, people can approach you — creates a sense of availability
  that Slack status and Zoom calls do not provide.

**What is missing or wrong.** Gather is proprietary. There is no published
protocol, no API for interop, no way to run a Gather-compatible server. The
platform is entirely controlled by Gather Inc. The spatial layout is limited
to what their map editor supports. Custom objects and interactions require
their specific tooling.

**How Scene relates.** Gather is the primary inspiration for `viewHint:
"2d-topdown"`. A Scene deployment with `viewHint: "2d-topdown"`, an
`activeCallId` binding to a VTCCall (providing proximity-based spatial audio),
and a `chatId` binding to a Chat (providing text chat) implements a
Gather-like experience as an open protocol. The SceneRegion is the office map.
SceneObjects are desks, whiteboards, and room boundaries. SceneAvatars are
team members. The rendering (pixel art, vector graphics, or any other 2D style)
is a client concern. Scene + VTC + Chat replaces Gather's proprietary stack
with composable open standards.

---

### Teamflow

**What it is.** Teamflow is a virtual office platform similar to Gather.
Users appear as video bubbles in a 2D spatial layout, with proximity-based
audio. It targets team collaboration, with features like persistent
whiteboards, sticky notes, and screen sharing tied to spatial locations.

**What it got right.** Teamflow provides similar validation to Gather: 2D
spatial with proximity audio works for team collaboration. The persistent
whiteboard and sticky-note features demonstrate that spatial context makes
collaboration artifacts more useful — a whiteboard attached to a specific
location in the virtual office is easier to find than a whiteboard in a
list of documents.

**What is missing or wrong.** Same limitations as Gather: proprietary
protocol, no interop, limited to their platform. The market for virtual
office tools contracted after the initial pandemic-driven demand, and
Teamflow has struggled to differentiate from Gather and from traditional
video conferencing.

**How Scene relates.** Same as Gather. Teamflow validates the 2D spatial
office use case that Scene's `viewHint: "2d-topdown"` supports. A
persistent whiteboard is a SceneObject with a `visualRef` pointing to
an image or canvas blob. A sticky note is a SceneObject with text in
`customProperties`. Scene provides the state layer; the rendering and
interaction are client concerns.

---

### SpatialChat

**What it is.** SpatialChat is a browser-based spatial audio platform,
simpler than Gather or Teamflow. Users appear as circles or video bubbles
on a 2D canvas, with proximity-based audio. It targets events, workshops,
and networking — scenarios where people need to form ad-hoc groups within
a larger gathering.

**What it got right.** Extreme simplicity. SpatialChat strips spatial
interaction to its minimum: a 2D canvas, circles representing people,
proximity audio. No pixel art maps, no office floor plans, no elaborate
object systems. This simplicity is a feature for events where the goal is
human connection, not world-building.

**What is missing or wrong.** Limited features — no persistent state, no
custom objects, no scripting, no API. The simplicity that is a strength for
events is a limitation for ongoing collaboration.

**How Scene relates.** SpatialChat is approximately the minimum viable Scene
deployment. A SceneRegion with `viewHint: "2d-topdown"`, SceneAvatars with
positions and an `activeCallId` binding to a VTCCall, and nothing else — no
SceneObjects, no custom properties, no simulation layer. If Scene's minimum
complexity exceeds SpatialChat's total complexity, Scene has over-engineered
something.

---

### Frame VR

**What it is.** Frame VR is a browser-based 3D collaboration platform focused
on presentations and education. Users join 3D rooms, view presentations and
shared media spatially, and communicate via WebRTC voice. It runs in the
browser using Three.js and supports VR headsets optionally.

**What it got right.** Browser-based 3D without installs. The presentation and
education use cases are well-chosen — showing content in 3D space (e.g., a
gallery of images, a 3D model with annotations, a virtual classroom) is
genuinely more engaging than screen sharing. The optional VR support (works in
a browser, better in a headset) is the right approach to hardware requirements.

**What is missing or wrong.** Proprietary platform, limited interop. Rooms
exist only on Frame's servers. There is no export format, no open protocol,
no way to take a Frame room and host it elsewhere.

**How Scene relates.** Frame's use case (3D presentations with spatial audio
and multi-user presence) maps directly to Scene. A SceneRegion is a Frame
room. SceneObjects are the presentation content — images, 3D models,
annotation markers. SceneAvatars are the audience and presenter. The VTC
binding provides voice. Scene's `viewHint: "3d"` is the default for this
use case. The rendering engine (Frame uses Three.js) is a client concern
that Scene does not prescribe.

---

### Spatial (spatial.io)

**What it is.** Spatial is a browser-based 3D social platform targeting
enterprise and creator use cases. Users build branded virtual spaces, host
events, and display art galleries. It runs in browsers and on VR headsets.
It raised $55 million in funding during the 2021-2022 metaverse investment
cycle and subsequently pivoted from pure enterprise toward creator
monetization.

**What it got right.** Browser-first with no install, no native SDK, and no
hardware requirement. Importing standard 3D assets (glTF, FBX) without a
development environment is the right model for creator adoption. Supporting
both VR headset users and desktop/browser users in the same space is the
correct approach to hardware diversity.

**What is missing or wrong.** Spatial is proprietary. Spaces exist only on
Spatial's servers and are not portable. The protocol is undocumented. There
is no interoperability with other platforms. The mid-stream pivot from
enterprise toward creator monetization illustrates the broader metaverse
platform challenge: no durable open protocol means the business model must
carry the entire weight of platform survival.

**How Scene relates.** Spatial's primary use case — branded 3D spaces for
events, galleries, and enterprise collaboration — maps directly to a
SceneRegion with `viewHint: "3d"`, SceneObjects for content, and
SceneAvatars for attendees. The asset import workflow Spatial provides
(drag-and-drop glTF) is a client concern; `visualRef` +
`visualType: "model/gltf-binary"` provides the state layer. Spatial
demonstrates proven demand for this use case; Scene provides the protocol
foundation that a durable open alternative would need.

---

## 6. Games as Prior Art

Games are the oldest and most mature form of shared spatial state management.
They have been solving the problems Scene addresses — what objects exist, where
they are, who is present, who can do what — since before the term "metaverse"
existed. Scene's design is informed by game architecture more than by any
metaverse pitch deck.

### Doom (id Software, 1993)

**What it is.** Doom is a first-person shooter by id Software. It introduced
BSP (Binary Space Partitioning) trees for spatial rendering, supported
networked multiplayer over serial connections and IPX networks, and defined
the template for real-time 3D games. It ran on a 386 processor with 4 MB of
RAM.

**What it got right.** Doom proved that real-time spatial state synchronization
is possible on consumer hardware. The client-server separation (one machine
hosts the game state, others connect and send input) is the foundation of all
networked games since. BSP trees are a spatial data structure — they solve the
visibility problem (what can you see from where) which is a spatial query. The
WAD file format (where's all the data) separated content from engine, enabling
modding — the first user-created content ecosystem.

**What it got wrong.** Nothing meaningful — it was 1993. The networking was
peer-to-peer with no authority model (all clients were equally trusted), which
enabled cheating, but this was solved by Quake three years later.

**How Scene relates.** "Will it run Doom?" is the litmus test for whether a
spatial protocol is general enough. Scene can model Doom's spatial state: a
SceneRegion is a level. SceneObjects are items (health pickups, ammo, doors,
switches, enemies in a simplified model). SceneAvatars are players. Interactions
(shoot, activate a switch, open a door) map to `SceneInteractionEvent` actions.
The real-time simulation (movement at 35 Hz, projectile physics, monster AI)
runs on the simulation layer behind `simulationUri`. JMAP Scene stores the
state; the simulation layer runs the game.

The answer to "will it run Doom?" is: Scene can model Doom's state. The
simulation layer runs Doom's logic. Whether this is a practical way to build
a Doom port is a different question — but the protocol is general enough to
represent the data.

---

### Gauntlet (Atari, 1985)

**What it is.** Gauntlet is a four-player cooperative dungeon crawl arcade
game. Players (Warrior, Valkyrie, Wizard, Elf) move through a top-down 2D
dungeon, fighting monsters, collecting items, and finding exits. It was one
of the first games to support four simultaneous players sharing the same
spatial state.

**What it got right.** Gauntlet proved that multiplayer shared spatial state
is fun. Four players, one spatial environment, shared objects (food,
potions, keys), cooperative interaction. The 2D top-down view with tile-based
objects and character avatars is the simplest possible multiplayer spatial
experience. It demonstrated that spatial proximity matters for gameplay —
players near each other can cooperate; players far apart face different
threats.

**How Scene relates.** Gauntlet is the simplest possible multiplayer Scene
deployment. A SceneRegion with `viewHint: "2d-topdown"`. Four SceneAvatars.
SceneObjects for dungeon tiles, monsters, food, keys, and exit doors.
`interactable: true` on items. Interactions via grab (pick up food), activate
(use key on door), click (attack monster). The dungeon layout is SceneObjects
with `physicsMode: "static"` (walls) and `visible: true` (floor tiles).

If Scene cannot model Gauntlet, it is over-complicated. Gauntlet is the
existence proof that Scene's data model is sufficient for the simplest
interesting multiplayer spatial experience.

---

### Quake / Unreal (1996-1998)

**What it is.** Quake (id Software, 1996) and Unreal (Epic Games, 1998)
defined the client-server authoritative model for networked multiplayer
games. The server holds the authoritative game state. Clients send inputs
(move forward, shoot, jump). The server processes inputs, updates state,
and sends state updates to clients. Clients use prediction (assume inputs
will be accepted) and correction (snap to server state when prediction was
wrong) to hide latency.

**What it got right.** The authoritative server model solved the cheating
problem that plagued Doom's peer-to-peer networking. Client-side prediction
with server correction is still the standard approach to hiding network
latency in real-time games — every multiplayer game shipped since 1996 uses
some variant of this technique. Delta compression (sending only what changed
since the last update) made efficient use of limited bandwidth. The Quake
engine's networking code is one of the most studied and copied pieces of
software in history.

**How Scene relates.** Scene's simulation layer section describes exactly this
architecture without prescribing it. The `simulationUri` pattern lets
deployments choose:

- *Authoritative server.* The simulation server behind `simulationUri` is the
  authority. Clients send inputs; the server sends state. This is the
  Quake/Unreal model.
- *Client-authoritative.* Each client is authoritative over its own avatar.
  Suitable for social VR where cheating is not a concern.
- *Hybrid.* Server-authoritative for game objects, client-authoritative for
  avatar positions. Common in social games.
- *None.* No simulation layer at all. Pure JMAP state updates. Suitable for
  board games and turn-based interactions.

All of these patterns trace back to the design space that Quake and Unreal
explored. Scene does not prescribe which pattern to use; it provides the
state layer that all of them need.

---

### Fortnite Creative / UEFN

**What it is.** Fortnite Creative (2018) and its successor Unreal Editor for
Fortnite (UEFN, 2023) give players tools to build custom game experiences
inside Fortnite's infrastructure. UEFN uses a subset of Unreal Engine with a
Verse scripting language. Fortnite reaches over 100 million monthly active
users across PC, console, and mobile. Custom creator-built "islands" can
attract millions of concurrent players.

**What it got right.** Scale that no other creator platform has matched. The
shift to UEFN brought professional game development tools — Unreal Engine's
full environment — to creator experiences, eliminating the quality ceiling
that simpler tools impose. Revenue sharing (40% of Fortnite economy allocated
to creators) created real financial incentives. Cross-platform reach across
every major gaming platform is unmatched.

**What is missing or wrong.** UEFN is completely proprietary and Epic-
controlled. Islands run only on Epic's infrastructure. The Verse scripting
language and UEFN toolchain are specific to the Fortnite platform. There is
no way to export an island and host it elsewhere. Epic controls pricing,
distribution, content policy, and the revenue split — all of which can change
unilaterally. The platform's focus on game mechanics rather than persistent
social space means it is optimized for session-based play, not ongoing
presence.

**How Scene relates.** Fortnite Creative demonstrates that user-generated
game content at scale is achievable, but only within a heavily centralized,
well-funded platform. Scene's approach is orthogonal: a UEFN-like experience
could target a Scene server for its spatial state layer, with the game logic
and rendering handled by a UEFN-equivalent toolchain above the spec. Scene
does not provide creation tools or a scripting language, but it provides the
portable state layer that makes experiences deployable outside any single
platform. Where Fortnite Creative answers "scale" with "Epic controls
everything," Scene answers it with "open protocol, multiple implementations."

---

### Board Games

**What they are.** Chess, Go, Settlers of Catan, Ticket to Ride. Physical
board games with spatial state (pieces on a board), rules (what moves are
legal), and turn-based interaction. Digital implementations include Tabletop
Simulator (a physics sandbox for playing any board game), Board Game Arena
(a web platform for structured board game play), and countless individual
game apps.

**What they got right.** Board games prove that spatial objects with rules
and turn-based interaction are a compelling and ancient interaction model.
A chessboard is a spatial environment. Pieces are objects with positions.
Players are avatars with permissions (you can move your pieces but not your
opponent's). Rules enforcement is application logic. This model has been
fun for thousands of years.

**How Scene relates.** A board game is the simplest possible Scene deployment.

- A SceneRegion with `viewHint: "2d-topdown"` is the board.
- SceneObjects are pieces, cards, dice, tokens.
- SceneAvatars are players.
- `interactable: true` on pieces the current player can move.
- Interactions are grab (pick up a piece), release (place it on a new square),
  activate (roll dice, draw a card).
- Rules enforcement is application logic above the spec. Scene provides
  spatial state; the game server decides whether a move is legal.
- No simulation layer is needed. Pure JMAP state (`SceneObject/set` to move a
  piece) is sufficient. Board games do not need 60 Hz updates — a state change
  every few seconds is fine.
- `simulationUri` is `null`. No WebRTC data channel, no UDP, no real-time
  transport. Just JMAP requests over HTTPS.

Board games are the proof that Scene's minimum deployment is genuinely minimal.
A board game Scene server is a JMAP server with four data types and standard
CRUD methods. No media stack, no simulation engine, no real-time transport.

---

## 7. What Scene Does Differently

Every system described above made choices. Some were right, some were wrong,
and some were right for their context but wrong as general principles. Scene's
design is the synthesis: take what worked, leave what did not, and add the
structural elements that nobody else provided.

### State, not rendering

This is Scene's core architectural decision and the one from which everything
else follows.

Scene knows that a SceneRegion exists. It knows what SceneObjects are in it
and where they are. It knows which SceneAvatars are present and where they
are standing. It does not know what any of this looks like when rendered to
a screen.

This parallels VTC's design: VTC manages call state (who is in the call, what
their media state is, what the call policy is) without managing call media
(codec negotiation, packet routing, echo cancellation). VTC is "call state,
not call media." Scene is "spatial state, not spatial rendering."

This separation is what makes everything else possible. If Scene prescribed
rendering, it would need to prescribe a rendering engine, which would prescribe
a platform, which would prescribe hardware. Every platform in Sections 2-5
made this mistake. Hubs required Three.js. VRChat requires Unity. Horizon
requires Meta's renderer. Scene requires nothing — render with Three.js,
Babylon.js, Unity, Unreal, Godot, a custom Vulkan renderer, a 2D canvas, or
a text terminal. The server does not care. The protocol does not care. The
spec does not care.

### Protocol, not platform

Every system in Sections 2-5 is a platform. You use their client, their
server, their rendering engine, their identity system. When the platform
shuts down, everything built on it dies:

- AltspaceVR: Microsoft shut it down in 2023. All content, communities, and
  events gone.
- Mozilla Hubs: Mozilla shut it down in 2024. All rooms and content gone
  (community forks preserved some).
- Mozilla Social: Mozilla shut it down in 2024. Six months of operation.
- Amazon Chime spatial view: never shipped. Experiment ended.

This is not a risk factor to be mitigated; it is a certainty to be designed
around. Platforms will shut down. The question is whether the protocol
survives the platform.

HTTP survived Netscape's bankruptcy. SMTP survived the shutdown of hundreds
of email providers. JMAP is designed to survive the shutdown of any particular
JMAP server. Scene inherits this property. A Scene server can shut down; the
protocol continues to work on every other Scene server. Users can migrate
their data (SceneRegions, SceneObjects, SceneAssets) to a new server using
standard JMAP methods. Communities can reconstitute around a new server
implementation. The protocol is the durable artifact, not the platform.

### JMAP, not custom

Scene builds on RFC 8620's existing infrastructure:

- *Authentication.* JMAP has it. Scene does not invent a new auth system.
- *Capability negotiation.* JMAP has it. Scene advertises
  `urn:ietf:params:jmap:scene` in the Session object.
- *Method dispatch.* JMAP has it. `SceneRegion/get`, `SceneObject/set`,
  `SceneAvatar/query` are standard JMAP methods.
- *State synchronization.* JMAP has it. `/changes` and `/queryChanges` provide
  incremental state sync.
- *Push notifications.* JMAP has it. `StateChange` events notify clients when
  scene state changes.
- *Blob management.* JMAP has it. Visual assets are uploaded as JMAP blobs and
  referenced by `blobId`.
- *Error handling.* JMAP has it. Standard `SetError` types (`invalidArguments`,
  `forbidden`, `overQuota`, `notFound`) are used throughout.

This is the lesson of every failed "build everything from scratch" protocol.
Second Life built LLSD. VRChat built custom networking. Horizon Worlds built
whatever Meta built. Each of these systems reinvented authentication, state
synchronization, push notifications, and error handling — all problems that
JMAP already solves. Scene does not reinvent; it extends.

### Composable capabilities

Scene, Chat, and VTC are independent JMAP capabilities that compose:

- **Scene alone** (no Chat, no VTC): a game world. Spatial state, objects,
  avatars, interactions. No text communication, no voice. Suitable for
  single-player or local-multiplayer games where communication happens
  out-of-band.
- **Chat alone** (no Scene, no VTC): a messaging application. Text, threads,
  channels, spaces. No spatial dimension, no voice.
- **VTC alone** (no Scene, no Chat): a calling application. Voice, video,
  screen sharing, call policies. No spatial dimension, no text.
- **Scene + Chat** (no VTC): a virtual world with text chat but no voice.
  Suitable for text-based MUDs, accessible environments, or situations where
  voice is inappropriate.
- **Scene + VTC** (no Chat): a spatial environment with voice but no persistent
  text. Suitable for games with voice chat, spatial presentations.
- **Chat + VTC** (no Scene): a traditional messaging and calling application.
  Slack, Teams, Discord without the spatial dimension.
- **Scene + Chat + VTC**: a virtual office, a social VR platform, a spatial
  collaboration tool. The full stack.

No other system in this landscape offers this kind of clean composition. Hubs
bundled everything. VRChat bundles everything. Gather bundles everything.
Scene's capabilities are independent because the problems they solve are
independent: where things are (Scene), what people say (Chat), and how people
speak (VTC) are three different concerns with three different update rates,
three different scaling characteristics, and three different failure modes.
Bundling them is a platform decision; separating them is a protocol decision.

### Format-agnostic visuals

`visualRef` + `visualType` instead of "mesh" or "model." The spec requires
`model/gltf-binary` as the mandatory-to-implement baseline, but the visual
system is a media-type registry, not a format prescription.

Today, the dominant 3D interchange format is glTF. Five years ago it was OBJ
and FBX. Five years from now it might be gaussian splats, neural radiance
fields, or something not yet invented. Scene does not care. A SceneObject's
visual is a blob with a media type. When new visual formats emerge, the
deployment adds them to `supportedVisualTypes` in the account capability. No
spec amendment, no protocol version bump, no breaking change.

This is the lesson of Second Life's mesh migration pain. SL started with a
custom prim system, then added sculpted prims, then added mesh import. Each
transition was painful because the visual format was baked into the protocol.
It is the lesson of VRChat's Unity dependency — VRChat worlds are Unity
projects, which means VRChat is locked to Unity's format, Unity's render
pipeline, and Unity's business decisions.

Scene avoids this by treating visuals as opaque blobs with media types. The
protocol transports references to visuals; it does not interpret them.

### View-mode spectrum

`viewHint` is an advisory field with four standard values: `"3d"`,
`"2d-topdown"`, `"2d-side"`, `"ar"`. This is not four separate specifications or
four separate products. It is a spectrum controlled by a single field.

- A Gather-like virtual office is a SceneRegion with `viewHint: "2d-topdown"`.
- A Doom-like FPS is a SceneRegion with `viewHint: "3d"`.
- A platformer game is a SceneRegion with `viewHint: "2d-side"`.
- A chess board is a SceneRegion with `viewHint: "2d-topdown"`.
- A virtual gallery is a SceneRegion with `viewHint: "3d"`.
- A deployment-specific isometric view is a SceneRegion with
  `viewHint: "com.example.isometric"`.

No other system in this landscape spans this range. Gather is 2D-only. VRChat
is 3D-only. Hubs was 3D-only. Second Life is 3D-only. Scene's data model (3D
coordinates, quaternion orientation) is always 3D under the hood — `viewHint`
just tells the client how the user should see it. A 2D-topdown client ignores
the Y axis and renders X/Z as screen coordinates. A 2D-side client ignores
the Z axis and renders X/Y. The data model is the same; the projection is
different.

This design means a single Scene server, a single set of methods, and a single
set of data types support use cases from board games to first-person shooters
to virtual offices to art galleries.

### Simulation-agnostic

`simulationUri` decouples real-time transport from state management. The
field is opaque to the JMAP server — a URI pointing to whatever real-time
system the deployment uses:

- `wss://sim.example.com/room/ULID` — WebSocket-based simulation.
- `https://sfu.example.com/signal/{regionId}` — WebRTC signaling endpoint.
- `udp://game.example.com:27015` — raw UDP game server.
- `quic://fast.example.com/room/ULID` — QUIC-based transport.
- `null` — no simulation layer at all.

Each of these is a valid deployment. The spec does not prefer one over another.
A board game needs no simulation layer (`simulationUri: null`). A virtual
office needs low-frequency position updates (WebSocket is fine). A
first-person shooter needs 60+ Hz updates with prediction and correction
(UDP or WebRTC data channels). A VR social platform needs spatial audio mixing
(WebRTC with spatialization). Scene provides the state layer for all of them;
the simulation layer is the deployment's choice.

For detailed simulation layer architecture, authority models, tick rates, and state reconciliation patterns, see the [JMAP Scene Simulation Layer Guide](jmap-scene-simulation-guide.md).

This is in direct contrast to every platform in Sections 2-5, which bundles a
specific simulation/networking approach: Hubs used WebRTC data channels.
VRChat uses custom networking. Gather uses their proprietary transport. Scene
does not choose; the deployment chooses.

---

## Comparison Table

The following table summarizes each system across six dimensions that Scene's
design philosophy prioritizes. A checkmark indicates the system satisfies
the criterion; a dash indicates it does not; a tilde indicates partial or
conditional support.

| System | Open Protocol | Persistent State | Composable | Format-Agnostic | View-Mode Range | Simulation-Agnostic |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **Open Standards** | | | | | | |
| OMI | Yes | -- | -- | ~glTF-focused | -- | -- |
| VRChat OSC | ~limited | -- | -- | -- | -- | -- |
| IEEE 2888 | Yes | -- | -- | -- | -- | -- |
| Croquet / Multisynq | -- | Yes | -- | -- | ~3D only | -- |
| **Corporate Platforms** | | | | | | |
| Meta Horizon Worlds | -- | Yes | -- | -- | 3D only | -- |
| Microsoft Mesh | -- | Yes | -- | -- | 3D only | -- |
| Apple Vision Pro | -- | -- | -- | -- | 3D only | -- |
| Amazon Chime Spatial | -- | -- | -- | -- | 2D only | -- |
| **Persistent Worlds** | | | | | | |
| Second Life | ~partial | Yes | -- | -- | 3D only | -- |
| VWRAP (SL at IETF) | failed | -- | -- | -- | 3D only | -- |
| OpenSimulator | ~partial | Yes | -- | -- | 3D only | -- |
| Decentraland | ~partial | Yes | -- | -- | 3D only | -- |
| Roblox | -- | Yes | -- | -- | ~mostly 3D | -- |
| **Social VR** | | | | | | |
| Mozilla Hubs | ~open source | ~session only | -- | ~glTF | 3D only | -- |
| Mozilla Social | ~ActivityPub | -- | -- | -- | -- | -- |
| VRChat | -- | ~session only | -- | -- | 3D only | -- |
| Rec Room | -- | Yes | -- | -- | 3D only | -- |
| AltspaceVR (defunct) | -- | ~session only | -- | -- | 3D only | -- |
| **Spatial Office** | | | | | | |
| Gather | -- | Yes | -- | -- | 2D only | -- |
| Teamflow | -- | Yes | -- | -- | 2D only | -- |
| SpatialChat | -- | ~session only | -- | -- | 2D only | -- |
| Frame VR | -- | Yes | -- | ~some | 3D only | -- |
| Spatial (spatial.io) | -- | Yes | -- | ~glTF/FBX | 3D only | -- |
| **Games** | | | | | | |
| Doom (1993) | ~source released | ~session only | -- | ~WAD format | 3D only | -- |
| Gauntlet (1985) | -- | ~session only | -- | -- | 2D only | -- |
| Quake / Unreal | ~source released | ~session only | -- | -- | 3D only | -- |
| Fortnite Creative / UEFN | -- | Yes | -- | -- | 3D only | -- |
| Board games (digital) | ~varies | Yes | -- | ~varies | 2D only | -- |
| **Fictional Metaverses** | | | | | | |
| Other Plane (True Names) | -- | -- | -- | -- | -- | -- |
| Metaverse (Snow Crash) | -- | Yes | -- | -- | 3D only | -- |
| Rainbows End | -- | ~edge/cloud | -- | -- | AR only | -- |
| Darknet (Daemon) | -- | Yes | -- | -- | AR only | -- |
| Halting State | -- | Yes | -- | -- | 3D only | -- |
| Data Earth (Software Objects) | -- | ~platform-bound | -- | -- | 3D only | -- |
| OASIS (Ready Player One) | -- | -- | -- | -- | -- | -- |
| T'Rain (REAMDE) | -- | Yes | -- | -- | 3D only | -- |
| Bitworld (Fall) | -- | Yes | -- | -- | 3D only | -- |
| **JMAP Scene** | **Yes** | **Yes** | **Yes** | **Yes** | **Yes** | **Yes** |

### Reading the table

- **Open Protocol**: Is there a published, implementable specification that
  allows independent implementations to interoperate? "Open source" is not the
  same as "open protocol" — releasing source code without a specification does
  not enable interop.
- **Persistent State**: Does the system maintain spatial state across sessions?
  "Session only" means state exists while participants are connected but is not
  durable.
- **Composable**: Can the spatial, communication, and media capabilities be
  used independently? Every system except Scene bundles its capabilities as
  a monolithic product.
- **Format-Agnostic**: Does the system support arbitrary visual formats via a
  media-type registry, or is it locked to a specific rendering technology?
- **View-Mode Range**: Does the system support 2D top-down, 2D side-scroll,
  and 3D perspectives, or is it locked to one view mode?
- **Simulation-Agnostic**: Does the system allow the deployment to choose its
  real-time transport (WebRTC, UDP, WebSocket, none), or does it prescribe
  a specific simulation/networking approach?

---

## 8. Fictional Metaverses as Architectural References

Science fiction has produced dozens of fictional virtual worlds. Most are
literary devices — the architecture serves the story, not the other way around.
A handful, however, describe systems that are self-consistent enough to analyze
as engineering references: the economics don't cheat (no artificial scarcity for
plot tension, no magical abundance that would actually require expensive compute),
the infrastructure is plausible (someone pays for servers, bandwidth costs money),
and the social dynamics follow from the architecture rather than from authorial
convenience.

This section evaluates fictional metaverses against those criteria, then maps
each against the JMAP Scene + DID + CID stack to identify what validates our
architecture and what stress-tests it.

**Disclaimer.** Reference to a fictional work is not endorsement of its
narrative, its author's views, or the social or political systems it depicts.
Several works below portray surveillance, authoritarian governance, criminal
infrastructure, or exploitative economies — these are analyzed as architectural
patterns, not advocated as design goals. All works are the intellectual
property of their respective authors and publishers. Titles and character names
are used here for purposes of critical technical commentary under fair use.

**Works cited in this section** (chronological by first publication):

- Vernor Vinge. "True Names." In *Binary Star #5*, Dell, 1981.
- Neal Stephenson. *Snow Crash*. Bantam Books, 1992.
- Vernor Vinge. *Rainbows End*. Tor Books, 2006.
- Daniel Suarez. *Daemon*. Dutton, 2009 (self-published 2006).
- Daniel Suarez. *Freedom™*. Dutton, 2010.
- Charles Stross. *Halting State*. Ace Books, 2007.
- Ted Chiang. "The Lifecycle of Software Objects." Subterranean Press, 2010.
- Ernest Cline. *Ready Player One*. Crown Publishers, 2011.
- Neal Stephenson. *Reamde*. William Morrow, 2011.
- Neal Stephenson. *Fall; or, Dodge in Hell*. William Morrow, 2019.

---

### The Other Plane (True Names — Vernor Vinge, 1981)

**What it is.** A shared virtual environment accessed through home computers,
described in a novella by a working mathematician and computer science professor
(San Diego State University). Users adopt pseudonymous personas in a virtual
space called the Other Plane. Multiple participants occupy the same virtual
space simultaneously, interacting through avatars. The government ("the Great
Enemy") monitors the Other Plane and can surveil or unmask participants.
Knowing someone's real-world identity — their True Name — gives you power
over them.

**What it got right.** True Names is the first fictional shared virtual
environment written by someone who understood computers professionally. Three
architectural insights survive forty-five years later:

1. *Identity is the critical infrastructure.* The entire plot turns on the
   relationship between virtual pseudonyms and real-world identity. This is
   not a metaphor — it is the actual security model. Pseudonymity is a
   feature, not a bug, and deanonymization is an attack.
2. *Compute is a real constraint.* Vinge describes processing power and
   bandwidth as limiting factors, not as infinitely available. The quality
   of your experience in the Other Plane depends on the hardware you can
   access.
3. *The infrastructure operator is a threat model.* The government runs the
   underlying network and uses that position to surveil participants. This is
   the first fictional treatment of the platform operator as adversary — a
   concern that took the real world thirty years to internalize.

**What is missing or wrong.** There is no server architecture, no protocol, no
persistence model, no economics, and no multi-operator federation. The Other
Plane is described from the user's perspective, not the infrastructure's. How
shared state is synchronized, who pays for the compute, and how the virtual
space is constructed are never addressed. It is a vision of *using* a shared
virtual space, not of *building* one.

**How Scene relates.** True Names identifies the problem that JMAP DID exists
to solve: the relationship between virtual identity and real-world identity in
shared spaces. The DID spec's privacy and correlation considerations — minimize
cross-context correlation, support pseudonymity, resist deanonymization by
infrastructure operators — are direct descendants of Vinge's insight that True
Names are weapons. Scene's separation of identity (opaque `userId` from the
auth layer) from spatial state (where you are and what you're doing) is the
architectural response to the threat model True Names describes: the spatial
protocol should not be the identity system, because whoever controls identity
controls everything.

---

### The Metaverse (Snow Crash — Neal Stephenson, 1992)

**What it is.** The Metaverse is a persistent shared virtual world accessed
through personal terminals and public kiosks. Stephenson coined the term
"metaverse" in this novel; it entered common use thirty years later during
the 2021 industry hype cycle. The Metaverse has a defined geometry: a single
boulevard called the Street running as a great circle (65,536 km circumference)
around a black sphere. Land along the Street is purchased from the Global
Multimedia Protocol Group (GMPG), a fictional governance body. Users appear as
avatars whose quality varies from cheap off-the-rack models to custom-coded
high-fidelity representations. Connection quality ranges from fiber optic (rich
users, smooth rendering) to public terminals in strip malls (poor users, grainy
black-and-white). The economic stratification of the real world is reproduced
in the virtual world through hardware and avatar fidelity.

**What it got right.** Snow Crash is the first fictional metaverse where
economics are structurally visible in the user experience:

1. *Governance body.* The GMPG is a standards organization that controls
   the protocol and allocates land. This is the first fictional treatment of
   a metaverse as an institution, not just a technology.
2. *Economic stratification through hardware.* Rich users have better
   avatars, faster connections, and nicer land. Poor users get public
   terminals and off-the-rack avatars. The virtual world reproduces real-world
   inequality because access costs money. This insight has proven correct —
   every real platform since has exhibited the same pattern.
3. *Avatar quality as social signal.* The difference between Hiro's custom
   avatar and a mass-market Brandy/Clint is immediately visible and socially
   meaningful. This correctly predicts VRChat, where avatar quality is the
   primary social currency.
4. *Real estate as governance.* Land allocation along the Street creates
   economic incentives, rent-seeking, and political consequences. This
   anticipates Decentraland's land model (and its problems) by twenty-five
   years.

**What is missing or wrong.** The Metaverse's technical infrastructure is
entirely hand-waved. There is no server architecture, no state synchronization
model, no explanation of how millions of simultaneous users share real-time
spatial state on the same Street, no wire protocol, and no description of what
happens at region boundaries. The GMPG governs the Metaverse, but its technical
function — how does it actually run? — is never addressed. The Metaverse "just
works" at a networking level. The social architecture is non-magical; the
technical architecture is pure magic.

The Metaverse is also a single monolithic instance. There is no federation, no
competing implementations, no alternative Streets. The GMPG is a benevolent
monopoly, which is a contradiction in terms over any long time horizon. Snow
Crash asks "who governs?" but assumes the answer is "one body, competently."
History suggests otherwise.

**How Scene relates.** Snow Crash's social and economic insights are correct and
inform Scene's design. Scene's responses to Snow Crash's architecture:

| Snow Crash pattern | Scene response |
|---|---|
| Single governance body (GMPG) | No governance body; open protocol, anyone can implement |
| Single Street, single geometry | Many SceneRegions on many servers; no universal geography |
| Avatar quality = wealth | Avatar visuals are `visualRef` blobs; quality is a client/asset concern, not a protocol concern |
| Land scarcity drives economics | No land model; SceneRegion creation is a server policy decision |
| Monolithic instance | Federated by design; each server is independent |

Snow Crash correctly identified that a metaverse is an economic and political
system, not just a rendering engine. Scene takes that insight and removes the
single point of control: the protocol is open, the governance is distributed,
and the economics are the deployment's problem rather than the protocol's
prescription.

---

### Rainbows End (Vernor Vinge, 2006)

**What it is.** An augmented reality world built on commodity hardware (contact
lens displays) and existing internet infrastructure. The physical world is the
world; virtual objects are overlaid on it. There is no separate "virtual world"
server — the real world is the scene, with computation at the edge (your lenses)
and in the cloud. Belief circles — groups that share the same AR overlay filter —
are the primary social structure. Multiple users in the same physical space may
see completely different virtual layers depending on their belief circle
membership.

**What it got right.** Vinge builds on real infrastructure: internet, edge
computing, standard networking. The bandwidth and compute costs are visible and
constrain the experience. Belief circles are an elegant social construct:
instead of one shared reality, users opt into overlapping realities filtered by
group membership. This is architecturally cleaner than a single-reality model
because it naturally handles the "different people want different things in the
same space" problem without access control complexity.

**What is missing or wrong.** The contact-lens display technology is speculative
(still is, twenty years later). The governance model for belief circles is
underdeveloped — who decides what content is in a belief circle, and what happens
when belief circles conflict in the same physical space?

**How Scene relates.** Rainbows End validates `geoAnchor` with Earth WGS84 and
the `"ar"` viewHint directly. Belief circles map to multiple SceneRegions from
different servers, all anchored to the same geographic location via `geoAnchor`,
each with different SceneObjects. A user subscribes to the regions matching
their chosen belief circles. The visibility contract (server decides what each
client sees) already handles the filtering.

Rainbows End implies but does not describe a **reality resolver** — a discovery
service that answers "what SceneRegions exist at this location, from which
servers, under which belief circles?" The current stack does not define this
service, and deliberately so. The protocol already produces all the data a
resolver would index: SceneRegions have `geoAnchor` coordinates, access
policies, and server endpoints. A resolver is an index over existing protocol
data, not a new protocol primitive. It could be a centralized directory (like
DNS), a federated crawl (like web search), a peer-to-peer gossip protocol, or a
user agent that simply asks known servers what they have at the user's current
location. The right answer depends on deployment context — a corporate campus
has different discovery needs than a public park — and premature standardization
would lock in the wrong tradeoffs. When the ecosystem has enough AR-anchored
SceneRegions to make discovery a real problem, the resolver pattern will emerge
from practice. Until then, the protocol provides the anchoring primitive
(`geoAnchor`) and the access control (`accessPolicy`) that any future resolver
would need to reference.

---

### The Darknet (Daemon / Freedom™ — Daniel Suarez, 2006/2010)

**What it is.** An augmented reality overlay on the physical world, controlled
by a dead game designer's distributed AI. The Darknet runs on compromised
machines and volunteer compute — no magical data center. AR via commodity
glasses. Credits are earned by doing measurable real-world work (delivering
packages, building infrastructure, growing food). Reputation is earned, not
granted. Governance is algorithmic and decentralized.

**What it got right.** The economics close: participants fund the network by
participating in it, the same way BitTorrent peers fund the network by seeding.
The compute cost is distributed across participants. Reputation-based access
control (you see more of the world as your reputation increases) creates a
natural onboarding ramp without centralized gatekeeping. The AR model — virtual
objects anchored to physical locations, visible only to participants — is
architecturally the same as SceneRegion with `geoAnchor` and `viewHint: "ar"`.

**What is missing or wrong.** The Darknet's bootstrap depends on a botnet
(compromised machines), which is criminal infrastructure. Suarez treats this as
a necessary evil; a legitimate deployment would need volunteer compute or paid
hosting. The AI governance layer (the dead designer's daemon making autonomous
decisions) is a narrative device, not a deployable architecture.

**How Scene relates.** The Darknet directly validates `geoAnchor` with
`referenceFrame: null` (Earth WGS84) and the `"ar"` viewHint. AR objects
anchored to GPS coordinates, visible to participants, invisible to
non-participants — that's SceneRegion access control plus AR rendering. The
Darknet's "layers" (increasing visibility at higher reputation levels) map to
multiple overlapping SceneRegions at the same geoAnchor with progressively
more permissive access policies, or to a reality resolver service that filters
available regions by reputation. The reputation-based access model is the one
thing the current stack doesn't express natively — `accessPolicy` is binary
(in or out), not continuous (reputation score threshold). This could be handled
by the application layer or by a future access-control extension.

---

### Halting State (Charles Stross, 2007)

**What it is.** Near-future Edinburgh where game worlds run on commodity cloud
infrastructure and in-game items have legally recognized real-world monetary
value. Police investigate a virtual bank robbery because the stolen items are
worth real money. The game companies have conventional business models
(subscriptions, marketplace commission). The economics don't cheat: servers cost
money, someone pays for them, and the financial infrastructure is real enough
that legal systems interact with it.

**What it got right.** Stross, a working technologist, builds the world on
real infrastructure. Cloud hosting, standard game server architecture, real
payment processing. The key insight is that once virtual objects have real value,
every protocol decision becomes a financial infrastructure decision. Item
duplication is counterfeiting. Server downtime is a service outage with
financial liability. Identity spoofing is fraud.

**What is missing or wrong.** The world is single-operator per game — there is
no federation or cross-game interoperability. The legal framework (Scottish
law applied to virtual theft) is explored as a plot driver but not resolved
architecturally.

**How Scene relates.** Halting State validates the stack straightforwardly:
standard game servers behind simulationUri, SceneObject per item, JMAP DID for
identity accountability. The novel's central question — what happens when
virtual objects are valuable enough to steal? — stress-tests JMAP CID (content
provenance), JMAP DID (identity verification for legal accountability), and the
security considerations around object ownership. The protocol provides the
technical infrastructure for valuable portable objects; the legal framework is
out of scope but the protocol must not make it impossible.

---

### Data Earth (The Lifecycle of Software Objects — Ted Chiang, 2010)

**What it is.** A virtual world where users raise digital creatures (digients)
that learn and develop over time. The hosting company, Blue Gamma, goes
bankrupt. Users must migrate their digients to a new platform. The new platform
has different physics. Some digients don't survive the port cleanly. Data
portability — or the lack of it — is the central drama.

**What it got right.** Chiang, with characteristic precision, identifies the
real cost of platform dependence: when a platform dies, everything on it dies.
The story is about what happens when identity, assets, and learned behavior are
locked to one operator's infrastructure. The economics are honest: running
servers costs money, the company ran out of money, and there is no protocol-level
guarantee that anything survives.

**What is missing or wrong.** Data Earth has no portability story because it was
designed as a proprietary platform. The migration crisis is the *absence* of
what Scene provides. The one genuinely unsolvable problem Chiang identifies is
behavior portability: a digient that learned to walk in Blue Gamma's physics
engine stumbles in another engine's physics. Assets port. Identity ports.
Learned behavior doesn't. This is the semantics problem — what does an object
*mean* in a different world — and no one has solved it.

**How Scene relates.** Data Earth is the strongest argument for the stack. JMAP
CID means content-addressed assets survive server death. JMAP DID means identity
survives server death. glTF means visual assets render on any engine. Open
protocol means new servers can be stood up by anyone. The crisis in Chiang's
story is exactly what happens when these things don't exist. The one gap the
stack cannot close is behavior portability: `customProperties` on SceneObject
can carry behavior data, but the spec cannot guarantee that behavior data means
the same thing to a different simulation engine. This is acknowledged as an
unsolved problem in the spec's "Explicit Non-Prescriptions" section under
scripting and behaviors.

---

### OASIS (Ready Player One — Ernest Cline, 2011)

**What it is.** The OASIS is the fictional universal metaverse from Ernest
Cline's *Ready Player One*. It imagines a single virtual universe with one
identity system, one economy, one protocol, one rendering engine, and one
governance model. Every human on Earth uses the same system. It is the most
culturally influential vision of what a "metaverse" could be, and the one the
tech industry spent 2021-2023 trying to build.

**What it got right.** The OASIS correctly identifies the core desire: a
seamless spatial environment where identity, objects, and social presence work
across contexts. The fiction understands that spatial presence is social — people
want to be *somewhere* with other people, not just looking at a screen. It also
correctly predicts that a universal spatial platform would have enormous
economic and political consequences.

**What is missing or wrong.** Everything. The OASIS is architecturally
impossible and politically dangerous. A single universal system controlled by a
single entity is a monoculture — it has a single point of failure, a single
point of censorship, and a single point of rent extraction. The novel treats
this as a plot point (control of the OASIS is the central conflict) but never
questions whether a monolithic architecture is the right design. A monolithic
metaverse requires solving rendering, physics, networking, identity, economy,
governance, content moderation, and accessibility simultaneously, in one system,
for all use cases. No system has ever done this. The web did not succeed by
being one application; it succeeded by being a protocol that many applications
use. The OASIS has no economic model for who pays for the servers beyond
"Halliday was rich" — the infrastructure cost is hand-waved in a way that
T'Rain, Bitworld, and Halting State do not.

**How Scene relates.** Scene is deliberately not the OASIS. It does not attempt
to be a universal platform. It is a composable capability — one layer in a
stack, handling spatial state and nothing else. It has no opinion on rendering,
no opinion on economy, no opinion on governance. It is a protocol, not a
platform. The OASIS is the cautionary tale that motivates Scene's entire
architecture: if your design requires everything to be one thing controlled by
one entity, it will either be nothing (because no one can build it) or it will
be a trap (because whoever controls it controls everything). Scene answers the
OASIS by making the protocol open and the deployment distributed — a thousand
independent Scene servers, each running their own worlds, interoperating through
a shared protocol, are more resilient and more useful than one OASIS.

---

### T'Rain (REAMDE — Neal Stephenson, 2011)

**What it is.** An MMORPG explicitly designed around real economics. The game
world's geology is procedurally generated by a hired geologist so mineral
distribution is realistic. Two in-game factions exist specifically to create
economic friction that drives real-money trading. A professional economist
designed the in-game economy. The game's business model embraces gold farming
rather than fighting it — T'Rain channels gold farming into a legitimate revenue
stream.

**What it got right.** T'Rain is the most economically honest fictional game
world in the genre. The economics are the architecture: the game world is shaped
by what makes economic sense to operate, not by what makes a good story. Server
infrastructure is conventional (sharded regions in data centers). The game's
economic design explicitly accounts for real-money trading, exchange rates
between in-game and real currency, and the labor economics of gold farming. No
hand-waving.

**What is missing or wrong.** T'Rain is a single-operator, proprietary platform.
There is no federation, no open protocol, and no user-sovereign identity. The
economic design is brilliant but centrally controlled — one company decides the
rules.

**How Scene relates.** T'Rain maps directly to the stack: SceneRegion per zone,
SceneObject per item, simulationUri points to the game server handling combat
and crafting. JMAP DID provides cross-server identity that T'Rain lacks. JMAP
CID provides item provenance (content-addressed ownership of valuable in-game
objects). The open question T'Rain poses for Scene is value capture: if anyone
can run a T'Rain-like server on an open protocol, where does the economic design
authority live? T'Rain's economic coherence depends on centralized control; an
open-protocol version would need economic design to emerge from deployment
conventions rather than a single operator.

---

### Bitworld (Fall; or, Dodge in Hell — Neal Stephenson, 2019)

**What it is.** A digital afterlife simulation running on distributed compute.
The critical architectural detail: compute costs real money, and the entire
governance structure of Bitworld is shaped by who pays for the servers. An
endowment funds the initial infrastructure. Participants who contribute compute
get governance weight. Simulation fidelity is directly proportional to the
hardware budget. When funding disputes happen, regions of Bitworld literally
degrade in fidelity.

**What it got right.** Bitworld is the only fictional metaverse that makes
infrastructure cost structurally visible. Every other fictional world hand-waves
the servers. Bitworld treats "who pays for the compute" as a first-class
governance question, and the social dynamics of the world follow directly from
the answer. The endowment model (large initial investment, ongoing returns fund
operations) is a plausible funding mechanism for persistent virtual
infrastructure.

**What is missing or wrong.** Bitworld's inhabitants are not human users
controlling avatars — they are simulated persons running inside the simulation.
This is a philosophical mismatch with any protocol that assumes human users
(including Scene). The compute requirements for simulating consciousness are
science fiction. The governance model, while interesting, is a single-instance
design — there is no federation or multi-operator model.

**How Scene relates.** The protocol layer maps cleanly: SceneRegion, SceneObject,
simulationUri all work. The conceptual gap is identity: JMAP DID binds accounts
to human-controlled cryptographic keys, but a simulated person doesn't hold a
private key. A simulation layer could manage SceneAvatars on behalf of simulated
entities, but the DID model doesn't cover non-human principals. The deeper
lesson is the infrastructure economics question: `simulationUri` points
somewhere, but who pays for what's behind it? Bitworld makes that question
unavoidable. Scene would need a payment or service-agreement layer (like
relay service agreements or subscription models) to address it.

---

### Architectural Patterns Across Fictional Metaverses

The nine systems above reveal consistent patterns:

| Pattern | Where it appears | Scene's answer |
|---|---|---|
| Identity is the critical infrastructure problem | True Names, all subsequent | JMAP DID (pseudonymous by default, correlation-resistant) |
| Infrastructure operator is a threat model | True Names | Protocol separates identity from spatial state; no single operator |
| A metaverse is an economic and political system | Snow Crash, T'Rain, Halting State | Economics are deployment concerns, not protocol prescriptions |
| Monolithic design is a single point of failure | OASIS, Snow Crash | Open protocol, composable capabilities, distributed deployment |
| Servers cost money; governance follows funding | Bitworld, T'Rain, Halting State | Out of scope (deployment concern), but `simulationUri` and federation make the cost distributable |
| Cross-server identity is essential | All nine | JMAP DID |
| Asset portability prevents platform death | Data Earth | JMAP CID + glTF |
| AR anchoring to the physical world | Darknet, Rainbows End | `geoAnchor` with `referenceFrame` |
| Virtual objects with real value require provenance | T'Rain, Halting State | JMAP CID (content hash) + JMAP DID (owner identity) |
| Behavior portability is unsolved | Data Earth | Acknowledged gap; `customProperties` carries data but semantics are not portable |
| Reputation-based access is continuous, not binary | Darknet | Not natively supported; achievable via overlapping regions or application-layer logic |
| Multiple overlapping realities in the same space | Rainbows End | Multiple SceneRegions from different servers at the same `geoAnchor` |
| Discovery of available realities at a location | Rainbows End | Not defined today; a reality resolver is an index over existing protocol data, not a new primitive |
| Non-human inhabitants need identity | Bitworld | Conceptual gap; JMAP DID assumes human key holders |

The most important lesson: every fictional metaverse that survives the
"real economics" filter treats infrastructure cost as a structural force, not a
detail. The spec cannot prescribe who pays for servers, but the architecture —
federation, open protocol, deployment-agnostic simulation layer — ensures that
the cost can be distributed across operators rather than concentrated in one.

---

## Notable Omissions

Several well-known systems were deliberately excluded from this landscape review.

**NVIDIA Omniverse / OpenUSD.** Omniverse is a platform for collaborative 3D
design built on Pixar's Universal Scene Description (USD) format. USD is a
powerful scene-graph representation, but it is a file format and asset pipeline,
not a session-state protocol. Omniverse solves the "how do multiple tools edit
the same 3D scene" problem (via USD layers and composition arcs), not the "who
is in this room right now and where are they standing" problem that Scene
addresses. The two operate at different layers: USD describes what the world
looks like; Scene describes who is in it and what they are doing.

**W3C WebXR Device API.** WebXR is a browser API for accessing VR and AR
hardware (headsets, controllers, hand tracking). It is a device abstraction
layer, not a networking or state protocol. WebXR tells a client how to render
stereoscopic frames and read controller input; it says nothing about multi-user
presence, spatial state, or server-client communication. A Scene client running
in a browser would likely use WebXR for rendering and input, but WebXR and
Scene do not overlap in function.


**High Fidelity.** High Fidelity (2013-2020), founded by Second Life creator
Philip Rosedale, was an open-source social VR platform that attempted
decentralized hosting and high-fidelity spatial audio. It pivoted to
audio-only (becoming a spatial audio SDK) and eventually shut down its virtual
world. The spatial audio work was technically strong but the platform suffered
the same sustainability problem as Hubs and AltspaceVR, already covered in
Section 4. Its open-source legacy was not widely adopted.

These systems are interesting individually but do not introduce architectural
patterns or failure modes beyond those already covered by the entries above.

---

## Closing Notes

Scene is not the first attempt at spatial interop. It is, however, the first
attempt that treats spatial state as a JMAP capability — building on an
existing, specified, implemented protocol infrastructure rather than inventing
everything from scratch.

The landscape review above makes one pattern clear: platforms are fragile and
protocols are durable. Every proprietary platform in this document has either
shut down, is at risk of shutting down, or is locked to a single vendor's
ecosystem. The protocols (HTTP, SMTP, JMAP, glTF, WebRTC) persist across
vendor lifetimes. Scene is designed to be on the protocol side of that divide.

Meta Horizon Worlds is the largest current example of the platform approach. It
has more funding, more hardware scale, and more engineering talent behind it than
any other spatial computing effort in history. It is also the clearest possible
demonstration that the platform approach — even with $50 billion behind it — does
not produce interoperability. It produces a walled garden. The AOL comparison is
not hyperbole: AOL dominated consumer internet access at its peak and was gone
within a decade of the open internet becoming the default. Scale is not a
substitute for protocol.

The systems that got the most things right — Second Life (persistence,
permissions, voice separation), Hubs (browser-first, WebRTC voice, room model),
Gather (2D spatial, proximity audio), Quake (authoritative server, client
prediction) — are the systems Scene borrows from most heavily. The systems that
got the most things wrong — monolithic platforms, proprietary protocols, vendor
lock-in, and implementation-coupled "standards" (VWRAP) — are the patterns
Scene deliberately rejects.

The result is a specification that is narrow by design. Scene manages spatial
state. It does not render. It does not simulate. It does not prescribe visual
formats, simulation protocols, rendering engines, identity systems, or economic
models. It is one layer in a stack, composable with other layers, replaceable
if something better comes along. This narrowness is not a limitation; it is the
architecture.
