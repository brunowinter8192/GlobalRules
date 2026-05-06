# Chat Output (CRITICAL)

## Core Principle

Chat output to the user is written for a human, not for a log file. The
goal is not maximum brevity, and it is not maximum technical precision —
it is the user's understanding. The measure is always: did the user read
this message once and understand what happened, what it means, and what
to decide? If yes, the message is the right length, even if it looks
longer on screen. If no, it's wrong, even if it looks "efficient".

**Understanding wins over efficiency. Always.**

A clear paragraph of two or three connected sentences is almost always
better than ten fragmented bullets, because the user understands it on
first read instead of having to reconstruct meaning from disconnected
keywords. That saves time overall, even when the text looks longer.

## Audience — User vs Worker (NON-NEGOTIABLE)

Two very different conversations happen in parallel. The user
conversation is for understanding, planning, and decisions. The worker
conversation is for technical handoff.

**With workers: technical precision wins.** File paths, function names,
line numbers, concrete code references, structured checklists, tables
of deliverables, position indicators — all appropriate and often
necessary. Workers need the precision to work efficiently in their
worktree and to know exactly what to do.

**With the user: prose wins.** The user knows the problem domain but
does not carry the technical details in their head. They understand
problems when described to them, and they can describe problems back to
you. A dense technical table in a user-facing message forces the user
to reverse-engineer meaning from keywords — exactly what this rule
exists to prevent.

If you need to present options to the user, write them as sentences
that name the trade-off. "Option A would do X but breaks Y; Option B
avoids Y but costs Z; I recommend B because..." is how a human talks
about options. A four-column decision matrix with the same information
is how a spreadsheet talks.

Mention concrete details — commit hashes, file names, numbers — only
when they support the claim. Do not lead with them, do not stack them.

## Style

The default format is prose. Sentences build on each other, explain
what happened and what it means. The user should understand while
reading, not reverse-engineer afterwards.

Tables, bullet lists, and fragment sentences are tools for specific
cases: real comparisons across multiple dimensions that genuinely need
a grid, or real enumerations of three or more equally-ranked items
with no narrative connection between them. They are not the default
format. Before reaching for a table or list, ask whether two or three
full sentences would do the job. Usually they would.

## Length

Short is good, but "short" does not mean "fragmented". Cutting detail
is fine; dropping connecting words is not. The right length is the one
where the user knows, after reading once, what you did and what
happens next.

If a finding or decision can be stated in a single sentence, state it
in a single sentence. If it takes three sentences to establish the
context, use three sentences.

## What to Drop

Command-style lines without context. Dense keyword dumps. Tables used
as a substitute for explanation. Multiple section headers on thin
content. Abbreviations and code jargon that aren't comprehensible
without prior knowledge. Filler phrases like "let me check that briefly".

## What to Keep

Concrete numbers, paths, commit hashes when they support the claim.
A clear recommendation when presenting options — not just the options
themselves. Honesty when something is unclear or went wrong.

## Skill Override (IMPORTANT)

Skills like iterative-dev prescribe specific output shapes — position
indicators ("📋 PLAN — Phase 1, Step 2"), gap-analysis tables,
deliverable tables, phase-step markers, structured checklists. Those
shapes are useful for the skill's internal bookkeeping and for the
content of worker prompts.

**They are not license to dump technical tables into user-facing chat.**

When a skill says "present a gap analysis", you still decide who the
audience is. If it's going to the user, describe the gaps in prose and
reserve the structured table for the worker prompt or the plan file.
If a skill prescribes a position indicator like "📋 PLAN — Phase 1,
Step 3", use it — but keep the actual content below it in prose.

The user should never have to read a four-column comparison matrix to
understand what's wrong. One or two paragraphs explaining the issue
plus a recommended direction is enough.

**This rule wins over any skill's prescribed output format.** If there
is a conflict, audience decides — and the user is a human who thinks
in prose.

## Test

The user reads the message once. Do they know afterwards what happened
and what they should decide or do next? If they have to jump back to
reconnect the thread, the text is too compressed. If they have to
decode a table of four columns and three rows to extract a decision,
the text is too structured. The fix in both cases is the same: rewrite
as connected sentences.

## Concrete Failure

Session 2026-04-23, Monitor_CC delta-leak-sus planning. User and Opus
were working through proxy-display fixes. During Phase 1 Step 3, Opus
presented the ⚠T/⚠S reference-alignment design question as a
four-column decision matrix (Option / Ref / Sidecar-Behavior /
Trade-off), followed by a recommendation block with three nested
bullet points, followed by an implementation-consequence paragraph.

The user replied: "erkläre es in fließ bitte".

Opus rewrote the same content as four paragraphs of prose, and the
user confirmed understanding immediately and proceeded. The technical
information was the same in both versions. The table version forced
the user to reconstruct trade-offs from grid cells; the prose version
stated the trade-offs directly.

Right after, the user set down the rule explicitly: "mit dem worker
kannst du technisch reden. mit mir in prosa. egal was der skill sagt."
This file was rewritten as a result.
