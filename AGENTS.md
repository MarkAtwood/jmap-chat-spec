# Agent Instructions — jmap-chat-spec

## Read first

Before doing any work:
1. Read `PROJECT.md` — project goals and scope
2. Read the specific draft or guide you are working on
3. Run `bd ready` — check for open issues

## Project structure

This is a spec-writing project. All deliverables are Markdown files. There is no code, no build system, and no test suite.

**Normative drafts** (`draft-atwood-*.md`): IETF Internet-Draft format (kramdown-rfc). Use BCP 14 keywords per the discipline in `PROJECT.md`. The drafts are the source of truth for all protocol behavior.

Twenty drafts are in the corpus, in five families (`README.md` lists each one):

- **Chat:** core `draft-atwood-jmap-chat-00` plus `-federation`, `-push`, `-wss`, `-filenode`, `-calendars`, `-tasks`, `-did`, `-mimi`
- **VTC:** `draft-atwood-jmap-vtc-00`, `draft-atwood-jmap-vtc-wss-00`
- **Scene:** `draft-atwood-jmap-scene-00`, `draft-atwood-jmap-scene-wss-00`
- **Graph:** `draft-atwood-jmap-graph-00` plus `-projection`, `-search`, `-edit`
- **Shared building blocks:** `draft-atwood-jmap-cid-00`, `draft-atwood-jmap-did-00`, `draft-atwood-jmap-object-uri-00`

Every draft except `-chat-federation` (extends the core capability) and `-object-uri` (a URI scheme) has its own capability URN. Each companion is optional.

**Implementer guides** (`jmap-*-guide.md`, plus `jmap-chat-compliance-tests.md` and `jmap-unified-client-architecture.md`): Non-normative companions. Keywords only as described under Restrictions below. When a guide describes behavior defined in a draft, the guide must be consistent with the draft.

## Upstream draft sources

Editor's copies of the IETF drafts this corpus cites, cloned in `~/GIT` (shallow, `--depth 50`). They are often ahead of the posted revision; read them to see changes coming, but cite only posted revisions from datatracker. Checked 2026-10-18.

| Local clone (`~/GIT/`) | Origin | Holds | Notes |
|---|---|---|---|
| `draft-ietf-mimi-content` | github.com/ietf-wg-mimi/draft-ietf-mimi-content | -09 | Detached at tag `draft-ietf-mimi-content-09` on purpose (the cited revision); `main` is ahead |
| `mimi-protocol` | github.com/ietf-wg-mimi/mimi-protocol | -06 | |
| `mimi-room-policy` | github.com/ietf-wg-mimi/mimi-room-policy | -04 | |
| `jmapio-jmap` | github.com/jmapio/jmap | calendars -33 | Fastmail-edited drafts under `spec/`: calendars, contacts, sharing, tasks |
| `draft-gondwana-jmap-filenode` | github.com/brong/draft-gondwana-jmap-filenode | -16 | Repo name predates WG adoption; file is `draft-ietf-jmap-filenode.mkd` |
| `draft-gondwana-jmap-conditional` | github.com/brong/draft-gondwana-jmap-conditional | -01 | Not cited; overlap check only |
| `draft-gondwana-jmap-object-history` | github.com/brong/draft-gondwana-jmap-object-history | individual -00 | Stale: posted WG -02 is not in this repo; its source location is unknown |
| `draft-gondwana-jmap-blobext` | github.com/brong/draft-gondwana-jmap-blobext | individual -06 | Stale: posted WG -02 is not here (branch `msater` is older too); source unknown |

No editor's repo was found for draft-ietf-jmap-metadata, -refplus, or -mail-sharing. MLS drafts live under github.com/mlswg; none are cited directly (RFC 9420 is final).

To check for upstream changes: `git -C ~/GIT/<clone> fetch` and diff, then confirm against the posted revision before changing a citation.

## Before editing any file

1. Read the file in full before changing anything.
2. If editing a guide, read the draft(s) it references to verify consistency.
3. If editing a draft, grep the guides for any text describing the behavior you are changing.

## Cross-file consistency

| If you change... | Also check... |
|---|---|
| A field name or method signature in a draft | Every guide that names that field (`grep -l`), plus `jmap-chat-compliance-tests.md` for Chat drafts |
| Suppression or rate-limit behavior in a draft | The guide section describing that behavior |
| A capability URN or error string | Every file that mentions it (`grep -r`) |
| Urgency values or their semantics | Urgency tables in `jmap-push-platform-guide.md` |
| Push payload structure | `jmap-chat-push-platform-guide.md` encoding and truncation sections |
| WebSocket event delivery rules | `jmap-chat-wss-guide.md` suppression and handling sections |
| Deployment-defined / impl-defined posture in the core draft | `jmap-chat-implementer-guide.md` for the topic's section |
| Calendar binding / RSVP / availability behavior | `jmap-chat-calendars-guide.md` |
| TaskList binding / Task-Chat back-reference / task workflow | `jmap-chat-tasks-guide.md` |
| Peer auth mechanism / allowlist / abuse mitigation / federation observability | `jmap-chat-federation-guide.md` |
| File storage backend / scanning / quotas / previews / grace-period implementation | `jmap-chat-filenode-guide.md` |
| DID method support / federation signature mechanism / DID resolution / DID document conventions | `jmap-chat-did-guide.md`, `jmap-did-guide.md`, `jmap-toip-mapping-guide.md`, `jmap-chat-vc-guide.md` |
| CID / blob digest behavior | `jmap-cid-guide.md` |
| `jmap:` object URI form or the Object URI Types registry | Every draft that cites the registry (`grep -l 'JMAP Object URI Types' draft-*.md`) and `draft-atwood-jmap-chat-federation-00.md` |
| VTC call state, recording, ring push, gateways | The matching `jmap-vtc-*-guide.md`; `jmap-vtc-chat-*-guide.md` if Chat links change |
| VTC or Scene WebSocket events | `jmap-chat-wss-guide.md` and `jmap-scene-wss-guide.md` interop sections, plus the draft's own guide |
| Scene data types, simulation, or visibility | `jmap-scene-*-guide.md` (implementer, simulation, 2d, board-games, vtc-integration) |
| Graph data types or companions | The other Graph drafts (projection, search, edit build on graph-00) |
| Which capabilities exist | `jmap-unified-client-architecture.md`, `README.md` |

Always `grep -r <term> . --include='*.md'` before and after a change to catch all occurrences.

## Subagent guidance

- Spawn subagents for parallel work on different files.
- Never spawn two subagents that edit the same file — serialize those.
- Each subagent should read only the files it needs; do not dump the full repo into a subagent prompt.
- For consistency checks between a draft and a guide, give the subagent both files explicitly.
- If a subagent hits the same error three times without progress, stop and escalate rather than retrying.

## Restrictions

- Do not commit or push without explicit user approval
- Do not use TodoWrite or markdown task lists — use `bd create` for all tracking
- Do not add fields, methods, or behaviors not present in the drafts unless explicitly directed
- RFC 2119 keywords (MUST, SHOULD, MAY, etc.) may be used in guide files, but each guide must include the standard "A note on RFC 2119 keywords" wrapper near its top so readers know the keywords carry implementer-guidance weight, not normative protocol authority. The drafts remain the only normative source of truth.

## Beads Issue Tracker

This project uses **bd (beads)** for issue tracking. Run `bd prime` for full workflow context.

```bash
bd ready              # Find available work
bd show <id>          # View issue details
bd update <id> --claim  # Claim work atomically
bd close <id>         # Complete work
```

**Beads is the only task and planning tool.** Do NOT use:
- TodoWrite / markdown TODO lists
- Scratchpad or audit files (`audit-*.md`, `plan-scratch.md`, or any similar throwaway planning file)
- MEMORY.md or any other markdown file as a knowledge store

The only permitted markdown planning artifact is a permanent `PLAN.md` design document
checked into the repo — not a scratchpad. Use `bd remember` for persistent knowledge and
`bd create` for all task tracking.
