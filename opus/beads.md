# Beads (Cross-Session Context)

**MCP Tools (iterative-dev server):**

| Tool | Params | Notes |
|------|--------|-------|
| `bead_list` | `status: "open"\|"closed"`, optional `repo` | Default: open |
| `bead_show` | `id`, optional `repo` | Returns description + comments in one call |
| `bead_create` | `bead_title, description`, optional `labels, repo` | Type is always "task". For knowledge beads: `labels="knowledge"` |
| `bead_comment` | `id, text`, optional `repo` | For cross-project: pass `repo="/path/to/project"` |
| `bead_close` | `id, reason` | Reason must explain WHAT was done |

## Task Management Hierarchy

- **Beads** (`.beads/`) - Cross-session (days/weeks/months)
- **Plan-File** (`.claude/plans/`) - Within a session (hours)
- **TodoWrite** - Within an iteration (minutes)

## Bead Types

| Type | Scope | Lifecycle |
|------|-------|-----------|
| **Work Bead** | One Arbeitsstream | Open until the topic is done |
| **Knowledge Bead** | Topic-specific, timeless | Open indefinitely |

- Work Bead: One per Arbeitsstream, NOT per session. Title = TOPIC. Description = zero-context snapshot.
- Knowledge Bead: Create with `labels="knowledge"`. Referenced by any work bead, never owned by one.
- Key Principle: Beads track TOPICS, not time.

## Bead Usage (NON-NEGOTIABLE)

**Session Start:**
1. `bead_list()` — check for existing work beads
2. If relevant bead exists → `bead_show(id)` (returns description + comments), then continue
3. If no relevant bead → `bead_create(title, description)` BEFORE starting work
4. When user names a specific bead: ASK "Ist das Bead noch aktuell?" BEFORE scoping
5. After reading: address each open item, bug report, or action item explicitly

**Before Research/Agents:**
- Read bead comments before launching research agents on bead-related topics
- Check if the research question has already been answered in bead history

**During Session:** No mid-session bead commenting. Focus on the work.

**Session End:** ONE comment with STAND block — what's done, what's open. Close ONLY if topic is fully resolved.

## Content Requirements

Every bead MUST be understandable WITHOUT the original session context.

**On Create:** Description answers: What? Why? Where? (repos, files affected)
- Bad title: "Fix bug" — Good: "Fix NaN in search scores"

**Comments:** Capture SESSION NARRATIVE, not just outcomes.
- Include: Decisions + WHY, Alternatives rejected, Research findings, Reasoning gaps
- Bad: "Implemented X, Y, Z." — Good: "Chose X over Y because Z. Rejected A (redundant with B)."

**Progress Tracking — Session-end STAND block (MANDATORY):**
```
STAND:
- DONE: Point X, Y
- OPEN: Point Z, W
- NEW: Point V (added because ...)
- DROPPED: Point U (reason)
- APPROACH: How the work was done (workflow, worker patterns, what worked). A fresh Claude must be able to replicate the approach without guessing.
```

**On Close:** Reason explains WHAT was done, not just "done".

**Path Rule:** ALL paths relative to PROJECT ROOT. Name each repo when multiple are affected.
