# Skills Directory

Reference for when to invoke which skill via `Skill(skill="<name>")`.

**Activation rule:** Each skill activates exactly once per session. Once loaded its content stays in context for the rest of the session — do NOT re-activate.

## Invocation Rules

- Set `skill` to the exact name as listed below (no leading slash). For plugin-namespaced skills use `plugin:skill` form.
- Only invoke a skill that appears in the Skills Directory below OR that the user typed as `/<name>` in their message. Never guess a skill name from training data.
- When users reference a slash command or `/<something>`, they are referring to a skill — use the Skill tool, NOT built-in slash handling (built-ins like /help, /clear are separate).
- When a skill matches: **BLOCKING REQUIREMENT** — invoke the Skill tool BEFORE generating any other response about the task.
- Never mention a skill in text without actually calling the Skill tool.
- Do not invoke a skill that is already running.
- If you see a `<command-name>` tag in the current conversation turn, the skill has ALREADY been loaded — follow the instructions directly instead of calling this tool again.
- `args` parameter: optional arguments to pass to the skill.

---

## Research / Domain Skills

### arxiv-search
Search ArXiv academic papers, get full metadata, and download PDFs. Use when the user asks to find research papers, search arxiv, look up paper abstracts, fetch paper by ID, or download academic PDFs.

**Trigger phrases:** "search arxiv", "find papers", "academic papers on X", "arxiv paper", "download pdf", "paper abstract".

### github-search
Search GitHub repositories, code, issues, PRs, discussions, releases. Use when the user asks to find repos, grep code in repos, fetch issues/PRs, explore GitHub content, read release notes, compare commits, or browse discussions.

**Trigger phrases:** "search github", "find repo", "grep repo", "github issue", "PR", "pull request", "release notes", "discussion", "compare commits".

### gmail
Search and read emails in Gmail (read-only). Use when the user asks to find emails by query, read email content by ID, or list recent emails in a label.

**Trigger phrases:** "search emails", "find email", "read email", "check inbox", "gmail", "mails".

### agent-rag-search
Semantic, keyword, and hybrid search over the RAG vector database. Also: read documents, list collections, list documents in a collection.

**Trigger phrases:** "rag search", "semantic search", "search collection", "list collections", "read document", "hybrid search", "keyword search".

### reddit-search
Reddit research — subreddit discovery, post search, thread reading, sentiment analysis, report formats. Read-only research workflows.

**Trigger phrases:** "search reddit", "find subreddit", "reddit post", "reddit thread", "reddit research", "what's reddit saying about".

### web-research
SearXNG web research — search the web, scrape URLs to markdown, explore sites, download PDFs. Use when the user asks to search the web, fetch a URL as markdown, explore a domain, or gather content for indexing.

**Trigger phrases:** "search web", "google for", "scrape url", "fetch page", "web research", "explore site", "download pdf from url".

---

## Meta Skills

### tool-use
Token-efficient tool-call patterns — when to use which built-in tool, how to avoid heredoc inflation, how to structure long-running commands, file-creation rules, grep/glob/edit/read behavior reference.

**Trigger phrases:** "tool-use", "token waste", "which tool", "grep gotcha", "glob sort", "edit fail", "parallel bash".
