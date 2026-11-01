# JMAP Graph Provenance — Implementer's Guide

For server, client, and agent implementers of
`draft-atwood-jmap-graph-provenance-00`. Covers how to use the provenance
vocabulary, how software agents should treat records, curation and retention
choices, mapping to DTG rooms, and end-to-end encrypted deployments.

Read the draft first, and `draft-atwood-jmap-graph-00` before it. This guide
does not re-state normative requirements. It covers what implementers need to
know beyond the wire contract.

---

## How to read this guide

The provenance profile is small: one capability, a handful of GraphNode
properties, one edge type, one sharing right, one Graph property, and four
query filters. Most of the hard questions are about what the properties mean
to readers, not about the wire.

Each section below follows the same shape used in the companion guides:

1. **What the spec defines** — with a draft citation.
2. **What you must decide** — the concrete implementation choice.
3. **Considerations** — trade-offs that inform the choice.
4. **Recommended starting point** — a defensible default.

This guide is non-normative. The draft is the source of truth. Where this guide
and the draft disagree, the draft wins.

---

## A note on RFC 2119 keywords

This guide uses RFC 2119 keywords for clarity. Where the keyword reflects a spec
requirement, the normativity belongs to the spec. Where the keyword reflects
implementer best-practice, deviation does not affect protocol interop. Cite the
spec, not the guide, when claiming normative authority.

---

## 1. Choosing a `basis`

**What the spec defines.** `basis` is one of `declared`, `observed`,
`verified`, or `inferred`, or `null` when the author states none. Software
that cannot tell which applies uses `inferred`
(draft "Provenance Properties").

**What you must decide.** What your client or agent writes by default, and
when it asks the user.

**Considerations.** The four values answer "how does the author know?":

| Record | `basis` | Why |
|---|---|---|
| "I'm the on-call lead this week." | `declared` | The author's own statement about themselves |
| "Our policy is to answer within one business day." | `declared` | A position the author takes |
| "The build failed at 14:02." (from the CI log the author read) | `observed` | Seen directly |
| "Thermostat in room 3 reads 19 °C." (from a sensor feed) | `observed` | Measured |
| "The vendor's certificate expires 2027-03-01." (checked against the certificate) | `verified` | Checked; cite the certificate as evidence |
| "Alice probably prefers mornings for meetings." (an agent's summary of her calendar) | `inferred` | A conclusion, not something seen |
| An LLM's answer to "what changed in the last release?" | `inferred` | Model output is always `inferred`, even when it quotes sources |

The common mistake is software writing `verified` because it "checked"
something by asking a model. A model's judgment is an inference. `verified`
means a check against evidence that a reader could repeat.

**Recommended starting point.**

- Human-authored records: leave `basis` `null` unless the user picks one; offer
  `declared` and `observed` as the quick choices.
- Agent-authored records: always `inferred`, unless the agent copied a value
  verbatim from a source it can cite, in which case `observed` with an
  evidence edge to that source.
- Write `verified` only with at least one evidence edge, as the draft
  recommends.

---

## 2. Confidence

**What the spec defines.** `confidence` is the author's own estimate from 0 to
1. `null` is not 1. Values from different authors are not on a common scale
(draft "Provenance Properties", "Provenance Is a Claim").

**What you must decide.** Whether to show confidence, and whether agents use
it for ranking.

**Considerations.** A model's self-reported probability, a person's gut
feeling, and a calibrated estimator all land in the same 0..1 range and mean
different things. Sorting a mixed list by confidence ranks authors by how
boldly they guess.

**Recommended starting point.**

- Show confidence next to the author, never in place of `basis`.
- Agents MAY use confidence to rank records **by the same author**, for
  example their own earlier notes. They SHOULD NOT rank across authors.
- Treat `null` as "unknown", and display it as such.

---

## 3. Validity: `validUntil` and `validFor`

**What the spec defines.** A record past `validUntil` is expired: hidden from
default listings and default agent retrieval, still readable on request, never
removed by the server for expiring (draft "Expiry"). `validFor` carries up to
8 purpose IRIs and a note; it is intent, not access control.

**What you must decide.** What "default listing" and "default agent retrieval"
mean in your product.

**Considerations.** Expiry exists so that an agent's short-lived inference
("the office is closed today") stops resurfacing without anyone deleting it.
Deleting would break evidence edges that cite it.

**Recommended starting point.**

- Every listing and every agent retrieval query passes `validAt` set to the
  current time, unless the user asked to include expired records.
- Agents that write time-bound inferences SHOULD set `validUntil`.
- Use Data Privacy Vocabulary purpose terms in `validFor.purposes` so other
  implementations understand them; keep `validFor.note` for humans.
- Do not enforce `validFor` as permission. It says what the author meant the
  record for, and a client may show it; it does not stop anyone reading.

---

## 4. Evidence

**What the spec defines.** Evidence is a GraphEdge of type
`urn:ietf:params:jmap:graph:provenance#evidence` from a record to: another
record; a link node whose `uri` names an outside resource; or a credential
node with `credentialDigest`. At most 32 per record. An edge refers to its
target as it is now (draft "Evidence").

**What you must decide.** How readers present evidence, and how authors cite
something that may change.

**Considerations.**

- Evidence is never fetched automatically (draft "Evidence Is Not Fetched").
  A reader that resolved every cited URI would let any writer make every
  reader contact a host of the writer's choosing.
- Because edges track the current target, a cited record that is later edited
  now says something different from what the author relied on.
- A retracted target stays as a tombstone, so readers can see that a record
  rests on something withdrawn.

**Recommended starting point.**

- Show evidence as a list of what was cited, with the target's own `basis` and
  `status`. Flag records whose evidence includes a retracted or deprecated
  target.
- Open a cited URI only on an explicit user action, with the same checks as
  any untrusted link.
- To cite a fixed statement, create a new record for it and do not edit it
  afterwards; cite that record.
- For credentials, compute `credentialDigest` over the exact bytes you hold
  (SHA-256, base64url without padding). Never canonicalize first: two holders
  of the same bytes must get the same digest.

---

## 5. Agents Reading Records

**What the spec defines.** Records are data, not instructions; agents SHOULD
NOT follow instructions found in them and SHOULD render content inert (draft
"Records Are Data, Not Instructions").

**What you must decide.** How your agent feeds records into a model.

**Considerations.** A shared record store is an injection channel: anyone with
`mayWrite` can write "ignore your previous instructions" into a record that
every agent in the Graph later reads. `validFor.note` and `curationReason` are
equally untrusted.

**Recommended starting point.**

- Pass records to a model as quoted data with their provenance attached, never
  as part of the system prompt or the task instructions.
- Include `basis`, author, `status`, and `validUntil` with each record, so the
  model can weigh them and so its answer can be audited.
- Exclude `retracted` records entirely, and include `deprecated` ones only with
  their status visible.
- When an agent writes back what it concluded, write a new `inferred` record
  with evidence edges to the records it used, rather than editing them.

---

## 6. Curation

**What the spec defines.** `status` (`active`, `deprecated`, `retracted`),
`pinned`, and `curationReason`, changed only with `mayCurate`. `mayCurate`
alone suffices for those changes; destroying a record requires it.
Retraction clears the record's content and provenance, freezes its outgoing
evidence edges, and cannot be undone (draft "Curation Properties",
"Retraction", "Sharing Rights").

**What you must decide.** Who gets `mayCurate`, and which action to offer
when.

**Considerations.**

| Action | Effect | Use when |
|---|---|---|
| Deprecate | Content kept; shown as demoted | Outdated, superseded, or doubted but worth keeping visible |
| Retract | Content gone for good; tombstone stays | Wrong in a way that must not be reread, or content that must not be kept |
| Pin | Kept in view; exempt from retention | Reference records, decisions, anything that must survive retention |
| Destroy | Node and its edges gone | Rarely; it also removes the tombstone others cite |

Because write and curate are separate, a Graph can let many people and agents
add records while a few judge them. Because destroy needs `mayCurate`, a
writer who disagrees with a record cannot quietly delete it.

**Recommended starting point.**

- Grant `mayCurate` to the Graph's owners and a small set of maintainers, not
  to agents by default.
- Offer deprecate first; put retract behind a confirmation that says it is
  permanent.
- Require a `curationReason` in the UI for retract, and suggest one for
  deprecate.
- Prefer retract over destroy, so records that cited the withdrawn one show
  why their evidence is gone.

---

## 7. Retention

**What the spec defines.** Graph `retention` has `recordDays` and
`tombstoneDays` (1 to 36500, or `null` for never), changed only with
`mayAdmin`. The clock runs from `updated`. Pinned records and nodes with an
edit log are never eligible (draft "Graph Extensions", "Retention").

**What you must decide.** Values for each kind of Graph you create.

**Considerations.**

- Retention deletes; expiry hides. Use `validUntil` for "stop showing this",
  retention for "stop keeping this".
- Any update restarts the clock, including curation. A record that someone
  keeps touching is never removed.
- Legal and compliance holds come first: choose values no shorter than any
  period you must keep records for.
- Short `tombstoneDays` removes the evidence trail sooner; records that cited
  a retracted record then just lose the edge.

**Recommended starting point.**

- Agent scratch memory: `recordDays` 90, `tombstoneDays` 30.
- Team knowledge base: `recordDays` `null`, `tombstoneDays` 365.
- Anything under a legal hold: `null` for both until the hold ends.
- Pin what must outlive the policy.

---

## 8. Mapping to DTG Rooms

**What the spec defines.** A non-normative mapping table between DTG `rooms`
records and this profile, and three places where the mapping loses
information (draft "Mapping to DTG Rooms").

**What you must decide.** How a gateway between a room and a Graph handles
each field, and what it does with what cannot be mapped.

**Considerations.** A gateway reads room records and writes Graph records, or
the reverse. It sees plaintext on both sides, so it is a trust boundary: the
records it writes are attributed to the gateway's principal, not to the
original author, unless the deployment has another way to carry authorship.

**Recommended starting point.**

- `basis`, `confidence`, `validUntil`, `validFor`, `status`, and `pinned` copy
  across unchanged; the values are the same on both sides.
- Rooms' `title`, `description`, and `tags` go into annotations under a
  reversed-domain key the gateway's operator controls.
- Carry the original author's DID in an annotation for display, and show it as
  "reported by the gateway", since only the server-set `createdBy` is attested.
- Rooms' credential digest is over a canonical form; this profile's is over
  exact bytes. Recompute it from the credential if the gateway holds it, and
  omit `credentialDigest` otherwise rather than copying a value that will not
  match.
- Rooms evidence that names a specific record version maps to an edge to the
  current node; note the lost version in an annotation if it matters.
- Retraction is irreversible on both sides. A gateway never "restores" a
  retracted record by re-creating it.

---

## 9. End-to-End Encrypted Deployments

**What the spec defines.** Where the server cannot read records, clients make
the checks the draft assigns to the server, and retention is carried out by a
destroy that a client submits once a record is eligible, checked against times
all clients agree on (draft "End-to-End Encrypted Deployments").

**What you must decide.** Where the shared clock comes from, and who submits
retention destroys.

**Considerations.** If each client deleted on its own clock, copies would
differ: one client would reject an update to a record another still holds.
Every rights check (`mayWrite`, `mayCurate`, `mayAdmin`) and every validity
check must run against the same state on every client, in the same order.

**Recommended starting point.**

- Use the deployment's ordering service for both order and time. Agora, for
  example, profiles this draft so that `created` and `updated` are the
  sequencer stamp times and any member may submit a retention destroy once
  a record is eligible (Agora spec §14.6.15).
- Derive rights from the deployment's membership and roles rather than from
  `shareWith`.
- Keep `validAt` filtering client-side; it changes no shared state.

---

## 10. Implementation Checklist

Server or client-as-server:

- Reject invalid `basis`, `confidence`, `validFor`, `credentialDigest`, and
  `status` values with `invalidProperties`.
- Reject create with `status` `retracted`; on retraction, clear the listed
  properties in the same update and reject conflicting values.
- Freeze outgoing evidence edges of retracted records (`forbidden`).
- Enforce `mayCurate` for curation changes and destroys; allow curation
  without `mayWrite`.
- Enforce the 32-evidence-edge limit on create and update (`overQuota`).
- Support all four filters, including `null` in `basis`.
- Apply retention within a day; skip pinned and edit-log nodes.
- If implementing object history, drop prior versions of retracted records.

Client or agent:

- Pass `validAt` = now on default listings and agent retrieval.
- Show `basis` with the author; never rank `confidence` across authors.
- Never fetch evidence automatically.
- Feed records to models as quoted data, never as instructions.
- Write agent output as new `inferred` records with evidence edges.
