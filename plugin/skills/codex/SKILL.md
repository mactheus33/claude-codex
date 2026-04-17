---
name: codex
description: This skill should be used when the user invokes "/codex", or asks to "end the session", "close out", "save handoff", "update project memory", "compact memory", "write session summary", or any phrase indicating the working session should be wrapped up. Reads current conversation, rolls the active project's HANDOFF.md and learnings.md into concise rolling digests, and writes them only after user approval.
version: 0.1.0
---

# Codex — Rolling Digest

Wrap a Claude Code working session by compacting project memory into a small, current, load-bearing digest. Prevent `memory/` files from growing unbounded while preserving lessons that still matter.

## Philosophy

Append-forever memory grows into noise. This skill enforces a **rolling digest**: each session end, memory is re-read, merged with today's work, and rewritten concise. Three principles:

1. **Small** — bounded size, fits in context without burning tokens.
2. **Current** — reflects latest state, not history. History lives in git.
3. **Load-bearing** — every line earns its place by preventing a future mistake or clarifying current state.

## Files Managed

Relative to the active project root:

- **`memory/HANDOFF.md`** — current state. Always overwritten.
- **`memory/learnings.md`** — curated lessons. Always overwritten, after merging old + new.

Global auto-memory (`~/.claude/projects/.../MEMORY.md`) is NOT touched by this skill — it has its own discipline.

## Workflow

Follow these steps in order when the skill triggers.

### Step 1: Confirm project scope

Identify the active project from the current working directory and conversation context. If ambiguous, or if the directory has no `memory/` folder and creating one is not obviously correct, ask the user to confirm the project and path before proceeding. Do not assume.

### Step 2: Read current session context

Scan the conversation for:
- What was accomplished (final states, passing tests, deployed changes).
- Non-obvious lessons learned — technical, business, or process.
- Structural evolutions (stack changes, architecture decisions, new integrations).
- What is in progress, what is next, what is blocked.

### Step 3: Load existing memory

Read (if present):
- `<project-root>/memory/HANDOFF.md`
- `<project-root>/memory/learnings.md`

If either does not exist, treat as empty and plan to create.

### Step 4: Compose new HANDOFF.md

HANDOFF.md is always overwritten. Use this exact structure:

```markdown
# Session YYYY-MM-DD — <project-name>

## Handoff
- <1 line: what was done, final state>

## Next steps
- <bullets, actionable, prioritized>

## Blockers
- <only if blockers exist — omit the section entirely otherwise>
```

Rules:
- One date per file. Previous HANDOFF content is replaced, not appended.
- No filler. If "Next steps" is empty, state "None." Do not invent items.
- Direct, declarative language. No hedging.

### Step 5: Compose new learnings.md (digest)

learnings.md is also overwritten, but only after merging old + new.

**Keep from existing learnings:**
- Non-obvious AND still actionable.
- Lessons that, if forgotten, would cause a repeat mistake.

**Drop from existing learnings:**
- One-off bug fixes already reflected in code.
- Lessons that became code convention (they live in the code now).
- Anything generic already covered by global identity files (e.g. `~/.claude/soul.md`, CLAUDE.md).
- Duplicate items — merge into one.

**Add from today's session:**
- New non-obvious learnings.

**Cap:** ~150 lines total. If exceeded, compress further by merging similar items.

Format:

```markdown
# Learnings — <project-name>

## <Category>
- <direct, 1-line lesson>. Why: <optional but useful for non-obvious items>.
```

Categories emerge organically from content (e.g. Security, Infra, Business, Performance). Do not create empty categories.

### Step 6: Show drafts, request approval

Before writing, display to the user:

1. The proposed new `HANDOFF.md` (full content).
2. The proposed new `learnings.md` (full content).
3. A diff summary: "Kept N items from existing learnings, dropped M, added K new."

Ask explicitly: "Approve to write?"

### Step 7: Write only on approval

- On approval: write both files using the Write tool (overwrite).
- On redirection: revise based on user feedback, show again. Do not write until explicit approval.
- On rejection: stop. Do not partially write.

## Rules

- **One project per invocation.** Never write to multiple projects' memory in one run.
- **Never append.** Always read → merge → overwrite.
- **No fabrication.** If the session did not produce a learning, leave the learnings file untouched. Empty is better than invented.
- **Cross-project learnings** go to the destination project's `memory/incoming-learnings.md`, never direct edit.
- **Language:** always write memory files in English. Headers, structure, and example bullets are English regardless of the session's working language.

## Example Output

### HANDOFF.md

```markdown
# Session 2026-04-17 — viabox

## Handoff
- Validated e2e tests for MVP upload features. 15/15 passed.

## Next steps
- Deploy to staging.
- Schedule security review with external auditor.

## Blockers
- Waiting on API key for third-party auth provider.
```

### learnings.md

```markdown
# Learnings — viabox

## Security
- File size bypass in upload pentest. Fixed via max upload sizing. Why: attacker could bypass the MIME check by spoofing the extension on a file that exceeded content-type checks only after the decompression stage.

## Infra
- Hetzner CX33 peaks at 85% CPU during Docker rebuilds. Why: instance sized for steady-state, not burst.

## Business
- Landing page conversion +40% when phone field is optional. Why: real estate leads resist giving phone upfront.
```
