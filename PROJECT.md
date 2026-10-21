# jmap-chat-spec — Project Plan

## What This Is

A corpus of individual Internet-Drafts (`draft-atwood-jmap-*-00`) defining JMAP ([RFC 8620](https://www.rfc-editor.org/rfc/rfc8620)) capabilities, together with non-normative implementer guides. The corpus has five families:

- **Chat:** the core real-time messaging protocol, plus federation, push, WebSocket, FileNode, Calendars, Tasks, DID, and MIMI companions
- **VTC:** video/voice call state and its WebSocket binding
- **Scene:** spatial (3D/2D) state and its WebSocket binding
- **Graph:** unopinionated shareable graphs, plus projection, search, and live-edit companions
- **Shared building blocks:** CID (blob SHA-256), DID (account-to-DID binding), and the `jmap:` object URI scheme

[README.md](README.md) lists every draft and guide. The deliverables are Markdown only. There is no code, build system, or test suite in this repo; the reference implementation lives elsewhere (crate-jmap).

## Goal

Drafts that independent implementers can build interoperable servers and clients from, ready to post to the IETF datatracker. The guides cover what the drafts deliberately leave to deployments.

## How Work Is Tracked

All open work is in beads: `bd ready` for the frontier, `bd show <id>` for detail. This file does not hold a task list. Durable design decisions are stored with `bd remember`; the ones that shape everyday edits are summarized below.

## Standing Decisions

- **BCP 14 discipline (drafts):** MUST/MUST NOT only where interop breaks without it. Good practice, security hygiene, and harm limiting use SHOULD or plain prose. The single exception is federation request-signature verification and sender identity binding, which stay MUST.
- **BCP 14 in guides:** guides keep their keywords and their "A note on RFC 2119 keywords" section. Fix a guide keyword only when it contradicts a draft.
- **Drafts are authoritative.** When a guide and a draft disagree, fix the guide (or file a bead if the draft is wrong).
- **Naming:** capability URNs live under `urn:ietf:params:jmap:` and are assigned only through the JMAP Capabilities registry. References to objects are `jmap:` object URIs, not URNs.
- **Citations:** cite only posted datatracker revisions. Editor's copies in `~/GIT` (listed in AGENTS.md) are for seeing upcoming changes.
- **Optionality:** every companion is optional and has its own capability. A deployment without a companion must still be fully functional.

## What NOT To Do

- Do not add fields, methods, or behaviours that are not in the drafts unless explicitly directed.
- Do not propose or draft Graph federation. Federation across owners is still an open research problem; cross-graph edges stay forbidden.
- Do not restate an existing RFC requirement as a new MUST; cite it instead.
- Do not use markdown TODO lists, scratch audit files, or MEMORY.md; use `bd create` and `bd remember`.
- Do not commit or push without explicit approval.
