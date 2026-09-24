---
name: commit-message
description: Write git commit messages in this team's established style — a plain imperative sentence, no conventional-commit prefix (no feat:/fix:/chore:), with a ticket reference in parens only when the change maps to an actual ticket number. Use this whenever drafting, suggesting, or reviewing a commit message for this repo, or when the user asks "what should the commit message be" / "write a commit message" / runs `git commit`.
---

# Commit messages, this team's style

This repo's history is a mix of conventional-commit prefixes (`feat:`, `fix:`) and
plain sentences left over from earlier phases of the project. Going forward, use
**plain imperative sentences, no prefix**. Don't reach for `feat:`/`fix:`/`chore:`
even though you'll see them in older commits — that convention isn't the one to
extend.

## Format

```
<Imperative sentence describing the change>[ (ticket NNN)]
```

- Start with a capitalized imperative verb: "Add", "Show", "Fix", "Remove" — not
  "Added", "Adding", or "This adds".
- One line. No body, no bullet points, even for changes that touch several files —
  this team keeps subjects self-contained rather than splitting into subject+body.
- Keep it a plain sentence describing *what changed*, not *why* (the diff and any
  linked ticket carry the why).
- No period at the end.
- Name the specific thing touched — a function name, a file, a skill, a doc —
  rather than describing the area in general terms. Someone scanning `git log`
  should be able to tell *where* to look without opening the diff.

**Examples (real commits from this repo):**
- `Add dashboard date-range filter (ticket 005)`
- `Show each machine's specialty on its health card`
- `Fix allow SQLite connections across threads`
- `Add new drink type support to BrewOps`

**Being specific — compare:**
- Vague: `Update dashboard styling` → Specific: `Widen the grid gap in renderMachineCards`
- Vague: `Add skills` → Specific: `Add commit-message skill`
- Vague: `Fix bug in queries` → Specific: `Fix get_machine_health filtering out machines with zero brews`
- Vague: `Update docs` → Specific: `Document the ingest CSV filename prefixes in CLAUDE.md`

Pick the noun that actually identifies the change: a function/method name for code,
the skill's directory name for skills, the doc's filename for docs. If a change
touches several functions or files for one cohesive reason, name the most relevant
one or the shared concept (e.g. `renderMachineCards`) rather than listing all of
them — the goal is a precise pointer, not an exhaustive index.

## Ticket references

Append `(ticket NNN)` only when the change actually maps to a numbered ticket —
most commits don't have one and shouldn't invent a suffix. Only add it when:
- the user gives you a ticket number directly, or
- the work clearly originated from a ticket file already in the repo (e.g.
  `tickets/005-*.md`) that you can point to.

If there's no ticket, just leave the sentence as-is — an absent ticket ref is the
normal case, not a gap to fill.

## Before drafting

Look at the actual diff (`git diff --staged`, or `git diff` if nothing's staged)
rather than working from memory of what you changed — the message should describe
what's really in the diff, not the original ask. If a change is genuinely two
unrelated things, prefer splitting into two commits over cramming both into one
run-on sentence.
