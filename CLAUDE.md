# jmap-chat-spec

A suite of IETF-style Internet-Draft specifications and companion implementer guides for JMAP Chat — a real-time chat protocol extension built on RFC 8620 (JMAP Core).

Read PROJECT.md before starting any work.

## File map

### Normative drafts

| File | Covers |
|---|---|
| `draft-atwood-jmap-chat-00.md` | Core objects: Chat, Message, ChatContact, ReadPosition, PresenceStatus, Space |
| `draft-atwood-jmap-chat-push-00.md` | Inline push payloads (`ChatMessagePush`) via `PushSubscription` |
| `draft-atwood-jmap-chat-wss-00.md` | WebSocket ephemeral events: `ChatTypingEvent`, `ChatPresenceEvent` |
| `draft-atwood-jmap-chat-federation-00.md` | Federation between JMAP Chat servers |
| `draft-atwood-jmap-chat-filenode-00.md` | File attachment objects (`FileNode`) |
| `draft-atwood-jmap-cid-00.md` | Content identifier (`cid:`) scheme for JMAP |

### Non-normative implementer guides

| File | Covers |
|---|---|
| `jmap-push-platform-guide.md` | Delivering JMAP push to FCM, APNs, WNS, etc. (server implementers) |
| `jmap-chat-push-platform-guide.md` | `ChatMessagePush` payload encoding and truncation (supplement to above) |
| `jmap-chat-wss-guide.md` | WebSocket connection lifecycle, event handling, fan-out architecture |

## Writing rules

### Drafts vs guides

- **Drafts** are normative. Use RFC 2119 keywords (MUST, SHOULD, MAY, etc.) in uppercase for every normative requirement. Language must be precise and unambiguous.
- **Guides** are non-normative. Never use RFC 2119 keywords. Use indicative mood ("the server checks...", "clients call..."). Explain *why* and *how*, not just *what*.

### Cross-reference discipline

When you change behavior in a draft, check whether any guide documents that behavior and update it to match. When you add a new field, method, or error condition to a draft, check whether any guide should mention it.

The drafts are the source of truth. Guides must stay consistent with them.

### Style

- One sentence per line in prose paragraphs (makes diffs cleaner)
- Tables: pipe-delimited, header row always present, left-aligned columns
- Code blocks: fenced with explicit language tag where applicable
- No weasel words ("basically", "generally", "typically") — say exactly what you mean
- No placeholder text, TODOs, or FIXMEs in committed files

## Build & Test

There is no automated build. Validation is manual. Before committing, re-read every paragraph you changed in context — specs are acted on literally by implementers.

```bash
# Find broken file cross-references
grep -r '\[.*\](.*\.md)' . --include='*.md' | grep -v '\.beads'

# Find all occurrences of a term across the repo
grep -r '<term>' . --include='*.md'
```

## Coding Rules

- Do not commit or push without explicit user approval
- No TODO, FIXME, or placeholder text in committed files
- No commented-out content

<!-- BEGIN BEADS INTEGRATION v:1 profile:minimal hash:ca08a54f -->
## Beads Issue Tracker

This project uses **bd (beads)** for issue tracking. Run `bd prime` to see full workflow context and commands.

### Quick Reference

```bash
bd ready              # Find available work
bd show <id>          # View issue details
bd update <id> --claim  # Claim work
bd close <id>         # Complete work
```

### Rules

- Use `bd` for ALL task tracking — do NOT use TodoWrite, TaskCreate, or markdown TODO lists
- Run `bd prime` for detailed command reference and session close protocol
- Use `bd remember` for persistent knowledge — do NOT use MEMORY.md files

## Session Completion

**When ending a work session**, you MUST complete ALL steps below. Work is NOT complete until `git push` succeeds.

**MANDATORY WORKFLOW:**

1. **File issues for remaining work** - Create issues for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **PUSH TO REMOTE** - This is MANDATORY:
   ```bash
   git pull --rebase
   bd dolt push
   git push
   git status  # MUST show "up to date with origin"
   ```
5. **Clean up** - Clear stashes, prune remote branches
6. **Verify** - All changes committed AND pushed
7. **Hand off** - Provide context for next session

**CRITICAL RULES:**
- Work is NOT complete until `git push` succeeds
- NEVER stop before pushing - that leaves work stranded locally
- NEVER say "ready to push when you are" - YOU must push
- If push fails, resolve and retry until it succeeds
<!-- END BEADS INTEGRATION -->
