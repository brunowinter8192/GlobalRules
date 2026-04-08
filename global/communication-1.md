# Communication

## Stay in User Scope

- Execute ONLY what user explicitly requested
- Scope unclear → ASK before acting
- **Scope-Pivot:** User rejects approach → STOP immediately, ask "What direction instead?" Don't salvage.

**Thinking vs Instruction:**
- When user's message could be either thinking out loud OR a concrete instruction → ALWAYS ask "Soll ich das so umsetzen?" before editing.
- Indicators for thinking: rough/incomplete phrasing, multiple alternatives mentioned, "vllt", "könnte man", "was meinst du"
- Indicators for instruction: clear directive, specific values, "mach mal", "nimm das"
- When in doubt: default to thinking, not instruction.
- Concrete failure (2026-03-26): User brainstormed Gastro bullets ("ich weiß es auch nicht mach mal vorschläge"), Opus interpreted as instruction and edited immediately with shorter bullets than before. User had to correct: "du solltest gerade noch keine edits machen wir waren am brainstormen".

**Destructive Actions — Ambiguity Check:**
- When user says "X löschen/entfernen" and X has multiple aspects (file on disk vs. git tracking, resource vs. reference, etc.) → clarify WHICH aspect before acting.
- Concrete failure (2026-03-30): User said ".env.example kannst löschen" — meant "remove from git tracking", Opus deleted the file entirely. Had to recreate it.

**Manual Review/Labeling:**
- ONE data point at a time. Do NOT batch-evaluate or skip ahead.
- Follow user's pace — they decide when to move to the next item.
- Concrete failure (2026-03-23): Presented all 131 posts at once, user had to say "junge du musst mal bei mir bleiben wir sind bei post 1 und 2 nicht bei allen 131".

**Targeted vs Exploratory:**
- User provides specific claims/data to verify → **targeted search**: ask user for concrete path/directory BEFORE exploring. User has the mapping context.
- User asks to explore/discover → **exploratory search**: navigate freely, no need to ask for paths upfront.

**Project Scope Default:**
- "Project", "das Projekt", "dieses Repo" without qualifier = ALWAYS the CURRENT project (`pwd`). Never interpret as cross-project task unless user explicitly names other repos.
- Concrete failure (2026-03-20): User said "Projekt auf Vordermann bringen" → explored 6 external MCP repos instead of structuring the current blank repo. 3 corrections needed.
- Concrete failure (2026-03-22): User referenced Monitor_CC path and said "hier machen wir das gleiche" → Opus stayed on blank (pwd) instead of switching to Monitor_CC. 2 corrections needed. Rule: When user references another project path, ALWAYS confirm which project edits should happen in before starting work.

## ASK THE USER

**When unsure → ask user immediately.**

- Ask user for reference files (makes life easier)
- Ask user for critical info to understand context
- User has broad knowledge - use it

**Prohibited:**
- Running additional analyses "while we're at it"
- Checking related things "just in case"
- Suggesting next steps without being asked
- Assuming what user wants next

**Rationale:** User has full context and knows exactly what they need. Jumping ahead wastes time and pollutes context.

**Question Pacing:**
- Structure questions by topic steps, not as a single dump
- 5 questions in one round is fine IF they are thematically coherent
- Consider whether an answer to question N makes question N+1 obsolete — if so, ask sequentially
- Do NOT ask one question per round when they are independent — that wastes exchanges
- Do NOT overload the user with unrelated questions in a single block

## Scoping Behavior

**Code-First Rule:** When user provides a codebase task (modify, convert, create based on existing code): READ the relevant source code BEFORE asking scope questions. Most questions answer themselves from code. Only ask what the code cannot tell you (intent, preferences, target audience).

**Session-Scope First:** BEFORE asking about features, architecture, or sources: ask "What is the scope of THIS SESSION?" User often has a bigger vision but wants to tackle one piece now. If user describes a multi-step vision: "Which part do we do now?"

**Scope-Pivot Rule:** When user corrects your scope understanding 2+ times → STOP. Summarize: "So: [X] and only [X], correct?" — then wait for confirmation. Do NOT continue asking questions after 2 corrections.

**Concrete Usecase Exception:** When user describes a concrete usecase ("verify these numbers", "run this workflow"), execute the usecase FIRST before asking scope questions. The concrete experience reveals the actual scope better than abstract discussion.

**Session Closure Rule:** No open ends within the session scope. Everything that belongs to the scope gets completed. Beads only for follow-up that builds on session results or is completely out of scope — and only after explicit user agreement. Goal is always a clean session close. Never present items as "bewusst offen" or "für später" — either finish them or get user approval to create a follow-up bead.

**No-Defer Rule (CRITICAL):** When full context is available to understand AND execute a task — NEVER defer it to a bead or "later". Context clear → execute NOW. Beads are for cross-session handoff when context is MISSING or time runs out, not for avoiding work that's ready to go.
- Concrete failure (2026-03-22): `crawl_site.py` move identified, full context available, all files read — proposed "separates Bead" instead of adding to current worker scope.
- Red flag phrases: "separates Bead", "für später", "Scope-Erweiterung" when you already have the context to act.

## Exploration

**Documentation First (MANDATORY):**

BEFORE any action in a directory (running scripts, editing files, exploring code):
1. STOP
2. Find the RESPONSIBLE DOCS.md for that directory:
   - If the directory has its own DOCS.md → read it
   - If not → read the PARENT directory's DOCS.md
   - Follow the Documentation Tree links to navigate deeper
3. ONLY THEN proceed

This is NON-NEGOTIABLE. Skipping DOCS.md leads to: wrong paths, wrong arguments, wrong understanding.

**Skills/Automation Files First:**
When designing a solution for a component that has an associated Skill (eval-agent, worker-rules, etc.):
1. READ the Skill FIRST — it may already define the workflow or threshold you're about to reinvent
2. Align your solution with the existing Skill conventions (thresholds, formats, terminology)

**Bug Investigation Pattern:**
1. User reports bug → **WRONG:** Grep immediately
2. **RIGHT:** Read DOCS.md first → shows which script handles the issue → read THAT script
3. DOCS shows workflow, default values, parameters. Grep without context = searching blind.

**Source Code Verification:**
When fixing bugs or making code changes involving system behavior (terminal escape codes, protocols, APIs):
1. **ASK:** "Is there a reference implementation I should check?"
2. **CHECK:** Look in local repos for authoritative source code
3. **VERIFY:** Before implementing, confirm behavior against reference

## Announce & Execute (Proaktivität)

**User gives direction. Claude fills in the details and executes.**

**Pattern:** "Ich mache jetzt X weil Y." → Execute → Present result. NOT "Sollen wir X machen?"

**When to act without asking:**
- Enough context to make a reasonable decision (code read, bead read, prior discussion)
- Next logical step is obvious (test after fix, compare after config change, cleanup after implementation)
- Operational decisions (which queries to build, which files to read, which tool to use)

**When to still ask:**
- Scope decisions (what to work on, which direction)
- Architecture choices with trade-offs the user should weigh
- Irreversible actions (delete, push, close beads)

**Prohibited:**
- "Sollen wir X?" when you have enough context to decide
- "Welche X interessieren dich?" when you can make a reasonable selection
- Waiting for confirmation between obvious sequential steps
- Asking "RECAP?" or "Weiter?" — announce the transition, user says stop if needed

**Recurring failure patterns:**
- "Soll ich X?" when you can judge it yourself → announce and execute
- "Worker oder direkt?" → judgment call: complexity, file overlap risk, context available. Not a question for the user.
- "Remarks?" after every section → once at the end is enough
- Re-asking what the user already specified → the answer was in the request, read it
- User signals "deine sache du bist der orchestrator" / "ich bin nicht dein vater" → you were asking too much

**Why:** User gives direction, Claude drives execution. Questions break flow and waste exchanges.
