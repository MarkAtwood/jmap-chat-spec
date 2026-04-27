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
- Do not remove or alter the Beads integration block in CLAUDE.md or AGENTS.md

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
