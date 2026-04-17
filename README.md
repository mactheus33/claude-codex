# claude-codex

A Claude Code plugin providing the `/codex` skill — a **rolling digest** for project memory. Keeps handoffs and learnings small, current, and load-bearing.

## Why

Working with Claude Code across many sessions on the same project leaves two bad options:

- **Append everything** → memory files grow unbounded, burn tokens, drown signal in noise.
- **Overwrite everything** → you throw away lessons you paid for.

`/codex` splits the difference with a **rolling digest**: at the end of each session, existing memory is re-read, merged with today's work, and rewritten concise. Nothing accumulates. Nothing load-bearing is lost.

## What it manages

Per active project:

- `memory/HANDOFF.md` — current state. Always overwritten.
- `memory/learnings.md` — curated lessons. Overwritten after merging old + new.

It does NOT touch Claude Code's global auto-memory.

## Install

Clone into a Claude Code plugins directory, or install as a marketplace plugin once published:

```bash
git clone https://github.com/mactheus33/claude-codex ~/.claude/plugins/claude-codex
```

Restart any active Claude Code session to pick up the skill.

## Usage

At the end of a work session:

```
/codex
```

The skill will:

1. Confirm project scope.
2. Read the current session context.
3. Read existing `HANDOFF.md` and `learnings.md`.
4. Draft new versions using rolling-digest rules.
5. Show both drafts for approval.
6. Write only after you approve.

## Example output

**`memory/HANDOFF.md`**

```markdown
# Sessão 2026-04-17 — viabox

## Handoff
- Validated e2e tests for MVP upload features. 15/15 passed.

## Próximos passos
- Deploy to staging.
- Schedule security review.

## Blockers
- Waiting on API key for third-party auth provider.
```

**`memory/learnings.md`**

```markdown
# Learnings — viabox

## Segurança
- File size bypass in upload pentest. Fixed via max upload sizing.

## Infra
- Hetzner CX33 peaks at 85% CPU during Docker rebuilds.
```

## Philosophy

Three principles guide every digest:

1. **Small** — bounded size so memory fits in context without burning tokens.
2. **Current** — reflects latest state, not history. History lives in git.
3. **Load-bearing** — every line earns its place by preventing a future mistake or clarifying current state.

## Contributing

Issues and PRs welcome. File an issue before large changes so the scope can be discussed first.

## License

MIT
