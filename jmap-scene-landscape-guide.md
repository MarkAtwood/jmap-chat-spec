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

### OASIS (Ready Player One)

**What it is.** The OASIS is the fictional universal metaverse from Ernest
Cline's 2011 novel *Ready Player One*. It imagines a single virtual universe
with one identity system, one economy, one protocol, one rendering engine, and
one governance model. Every human on Earth uses the same system. It is the most
culturally influential vision of what a "metaverse" could be.

**What it got right.** The OASIS correctly identifies the core desire: a
seamless spatial environment where identity, objects, and social presence work
across contexts. The fiction understands that spatial presence is social — people
want to be *somewhere* with other people, not just looking at a screen. It also
correctly predicts that a universal spatial platform would have enormous
economic and political consequences.

**What is missing or wrong.** Everything. The OASIS is architecturally
impossible and politically dangerous. A single universal protocol controlled by
a single entity is a monoculture — it has a single point of failure, a single
point of censorship, and a single point of rent extraction. The novel treats
this as a plot point (control of the OASIS is the central conflict) but the
tech industry spent 2021-2023 treating it as an aspiration. A monolithic
metaverse requires solving rendering, physics, networking, identity, economy,
governance, content moderation, and accessibility simultaneously, in one system,
for all use cases. No system has ever done this. The web did not succeed by
being one application; it succeeded by being a protocol that many applications
use.

**How Scene relates.** Scene is deliberately not the OASIS. It does not attempt
to be a universal platform. It is a composable capability — one layer in a
stack, handling spatial state and nothing else. It has no opinion on rendering,
no opinion on economy, no opinion on governance. It is a protocol, not a
platform. The OASIS is useful as a cautionary tale about monolithic design: if
your architecture requires everything to be one thing, it will be nothing.

---

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
Initially Quest-only, later expanded to mobile. Users can create worlds using
Meta's built-in tools, socialize in user-created spaces, and attend events.
It is the flagship product of Meta's "metaverse" pivot, which involved renaming
Facebook to Meta and investing tens of billions of dollars.

**What it got right.** Significant investment in spatial audio — proximity-based
voice in Horizon is genuinely well-implemented. Avatar expressiveness, despite
the initial lack of legs, has improved substantially. The creation tools are
accessible to non-developers. Meta's willingness to invest at scale in spatial
computing, even at enormous financial cost, pushed the industry forward.

**What it got wrong.** Horizon is a walled garden: Meta's client, Meta's server,
Meta's rendering engine, Meta's identity system, Meta's content moderation,
Meta's rules. There is no interoperability with anything outside Meta's
ecosystem. The "metaverse" branding triggered a cultural backlash that damaged
the broader spatial computing industry — people who might have been interested
in virtual worlds became skeptical because of the association with Meta's
corporate ambitions. User engagement numbers have been consistently
disappointing relative to the investment.

**How Scene relates.** Scene is the architectural opposite of Horizon. Open
protocol, no platform lock-in, no specific hardware requirement, no specific
rendering engine, no specific identity system. A Horizon-like experience could
be built on Scene + VTC + Chat, but so could a minimal 2D spatial office, a
board game, or a data visualization. Scene does not require a headset, does
not require a Meta account, and does not require anyone to say the word
"metaverse."

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
  became an IETF RFC or a W3C recommendation.
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

`viewHint` is an advisory field with three standard values: `"3d"`,
`"2d-topdown"`, `"2d-side"`. This is not three separate specifications or
three separate products. It is a spectrum controlled by a single field.

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
- `webrtc://sfu.example.com/room/ULID` — WebRTC data channels.
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
| OpenSimulator | ~partial | Yes | -- | -- | 3D only | -- |
| Decentraland | ~partial | Yes | -- | -- | 3D only | -- |
| Roblox | -- | Yes | -- | -- | ~mostly 3D | -- |
| **Social VR** | | | | | | |
| Mozilla Hubs | ~open source | ~session only | -- | ~glTF | 3D only | -- |
| VRChat | -- | ~session only | -- | -- | 3D only | -- |
| Rec Room | -- | Yes | -- | -- | 3D only | -- |
| AltspaceVR (defunct) | -- | ~session only | -- | -- | 3D only | -- |
| **Spatial Office** | | | | | | |
| Gather | -- | Yes | -- | -- | 2D only | -- |
| Teamflow | -- | Yes | -- | -- | 2D only | -- |
| SpatialChat | -- | ~session only | -- | -- | 2D only | -- |
| Frame VR | -- | Yes | -- | ~some | 3D only | -- |
| **Games** | | | | | | |
| Doom (1993) | ~source released | ~session only | -- | ~WAD format | 3D only | -- |
| Quake / Unreal | ~source released | ~session only | -- | -- | 3D only | -- |
| Board games (digital) | ~varies | Yes | -- | ~varies | 2D only | -- |
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

The systems that got the most things right — Second Life (persistence,
permissions, voice separation), Hubs (browser-first, WebRTC voice, room model),
Gather (2D spatial, proximity audio), Quake (authoritative server, client
prediction) — are the systems Scene borrows from most heavily. The systems that
got the most things wrong — monolithic platforms, proprietary protocols, vendor
lock-in — are the patterns Scene deliberately rejects.

The result is a specification that is narrow by design. Scene manages spatial
state. It does not render. It does not simulate. It does not prescribe visual
formats, simulation protocols, rendering engines, identity systems, or economic
models. It is one layer in a stack, composable with other layers, replaceable
if something better comes along. This narrowness is not a limitation; it is the
architecture.
