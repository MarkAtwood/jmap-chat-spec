# Agent Instructions — jmap-chat-spec

## Read first

Before doing any work:
1. Read `PROJECT.md` — project goals and scope
2. Read the specific draft or guide you are working on
3. Run `bd ready` — check for open issues

## Project structure

This is a spec-writing project. All deliverables are Markdown files. There is no code, no build system, and no test suite.

**Normative drafts** (`draft-atwood-*.md`): IETF Internet-Draft format. Use RFC 2119 keywords (MUST, SHOULD, MAY). The drafts are the source of truth for all protocol behavior.

**Implementer guides** (`jmap-*-guide.md`): Non-normative companions. No RFC 2119 keywords. When a guide describes behavior defined in a draft, the guide must be consistent with the draft.

## Before editing any file

1. Read the file in full before changing anything.
2. If editing a guide, read the draft(s) it references to verify consistency.
3. If editing a draft, grep the guides for any text describing the behavior you are changing.

## Cross-file consistency

| If you change... | Also check... |
|---|---|
| A field name or method signature in a draft | All three guides for references to that field |
| Suppression or rate-limit behavior in a draft | The guide section describing that behavior |
| A capability URN or error string | Every file that mentions it (`grep -r`) |
| Urgency values or their semantics | Urgency tables in `jmap-push-platform-guide.md` |
| Push payload structure | `jmap-chat-push-platform-guide.md` encoding and truncation sections |
| WebSocket event delivery rules | `jmap-chat-wss-guide.md` suppression and handling sections |

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
- Do not introduce RFC 2119 keywords (MUST, SHOULD, MAY, etc.) into guide files

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
