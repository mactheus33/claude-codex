# claude-codex

A Claude Code plugin with two skills for **project memory hygiene**:

- **`/codex`** — end-of-session wrap. Compresses the project's memory as a **rolling digest**: nothing accumulates, nothing load-bearing is lost.
- **`/otimizar-projeto`** — on-demand macro hygiene: large file splits, archiving by age, dangling-pointer sweeps.

## Why

Working with Claude Code across many sessions on the same project leaves two bad options:

- **Append everything** → memory files grow unbounded, burn tokens, drown signal in noise.
- **Overwrite everything** → you throw away lessons you paid for.

`/codex` splits the difference. Boot files stay small (word caps), history is preserved forever in an append-only changelog, and deep content lives in per-topic satellite files loaded on trigger.

## The memory model (two layers)

**Permanent layer** (project root, versioned like code):

| File | Cap | Behavior |
|---|---|---|
| `documento_mestre.md` | ~600 words | index: status + open items + rules + satellite map. **Replaced**, never stacked |
| `aprendizados.md` | ~800 words | curated lessons that still change future behavior. Garbage-collected |
| `changelog.md` | none | append-only history. **Never edited, never pruned** |
| `inbox.md` | none | cross-session mailbox (append blocks, consume on boot) |
| `<tema>_mestre.md` | <25KB | satellite: deep content per topic, loaded on trigger |

**Transient layer** (`memory/`): `handoff.md` (last-session state, overwritten each close) + scratch files triaged at session end.

## What `/codex` does at session close

1. Reads the session context and the current memory files.
2. Triages `memory/` scratch: promote to rules/lessons/satellites, log to changelog, or discard.
3. Appends a session entry to `changelog.md` (including the old handoff) — nothing is lost.
4. **Replaces** the master's status, GC's absorbed lessons, overwrites `handoff.md`.
5. Enforces the word caps, pruning in the same run if exceeded.
6. Verifies integrity before writing: no dangling file pointers, no pruning claims asserting an unverified destination.
7. Shows drafts for approval, writes only after you approve, then commits (and pushes) on `dev`.

## Install

From within Claude Code:

```
/plugin marketplace add mactheus33/claude-codex
/plugin install claude-codex
```

Restart the session to pick up the skills. Also works from a local checkout: `/plugin marketplace add <path-to-this-repo>`.

## Example output (session close)

```markdown
# Sessão 2026-04-29 — projeto-exemplo

## Handoff
- Refactor concluído: convenção de duas camadas aplicada. 39 rascunhos legacy triados.

## Próximos passos
- ZIP download de documentos
- Validar checkbox da landing page
```

…while the full detail of what was cut and why lands as a new entry in `changelog.md`, and the master keeps only live state.

## Philosophy

1. **Small** — boot files bounded in words, so memory fits in context without burning tokens.
2. **Current** — boot reflects latest state, not history. History lives in the changelog and git.
3. **Load-bearing** — every line earns its place by preventing a future mistake or clarifying current state.

## Contributing

Issues and PRs welcome. File an issue before large changes so the scope can be discussed first.

## License

MIT
