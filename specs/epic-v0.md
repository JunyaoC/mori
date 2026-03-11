# Epic: mori v0 — Proof of Concept

## Goal

A working system where every interaction (from any channel) becomes a GitHub Issue, the manager agent processes issues and acts, departments execute specialized work, and the agent can evolve its own system through PRs.

## Core Decisions

- **GitHub is the bus.** Every interaction creates or continues an issue. No exceptions.
- **DB is a search index**, not a source of truth. Rebuildable from GitHub.
- **Concierge has channel awareness.** Knows channel topology, routes correctly.
- **Manager is triggered by GitHub events.** Input = issue. Output = issue comments + actions.
- **Manager can self-evolve.** Modify prompt, tools, channels, knowledge, models — all via PRs.
- **Departments are autonomous.** Handle user input directly in department channels.

## Success Criteria

1. **Quick question** — "what's btc price" in #general → issue created, answered, closed in <6s
2. **Task dispatch** — "change footer to XYZ" → issue created, coding agent spawned, progress on issue, PR opened, #inbox notified on completion
3. **Bottom-up** — user in #project-x says "auth broken" → department handles directly, creates issue, fixes, PRs
4. **Cross-channel** — discuss topic in #general, reference it in #project-x → issues linked, context preserved
5. **Self-evolution** — agent learns preference → PRs to `.agent/knowledge/`, agent notices missing tool → PRs to `.agent/tools/`
6. **Cron reflection** — hourly journal, weekly self-review

## Milestones

### M1: Foundation

- [ ] GitHub Project Board (Backlog / In Progress / Review / Done)
- [ ] Label taxonomy: `quick`, `in-progress`, `needs-attention`, `needs-input`, `bug`, `incoming`
- [ ] `.agent/prompt.md` — manager personality
- [ ] `.agent/channels.json` — channel topology with types (concierge/department/inbox/journal)
- [ ] `.agent/models.json` — model tier configuration
- [ ] `.agent/tools/manifest.json` — tool registry
- [ ] SQLite DB schema (embeddings, thread_map, sessions, scratch)

### M2: Concierge

- [ ] Channel adapter contract: `listen()`, `post()`, `reply()`
- [ ] Channel registry (factory pattern, borrowed from NanoClaw)
- [ ] Channel-aware routing: read `.agent/channels.json`, route based on channel type
- [ ] Issue finding: check thread_map in DB → search GitHub → create new
- [ ] Message recording: every message → issue comment with `[source] @author` prefix
- [ ] Discord adapter: discord.py, message normalization, 2000-char chunking, typing indicators

### M3: Manager Core

- [ ] GitHub event handler (triggered by new issue comments)
- [ ] Context loader: issue + board + active tasks + related + semantic search
- [ ] System prompt loader: `.agent/prompt.md` + `.agent/knowledge/*.md` + `.agent/channels.json`
- [ ] Office tools: create_issue, update_issue, close_issue, add_comment, add/remove_label, move_card, link_issues, assign_issue, create_gist
- [ ] Communication tools: post_to_channel, notify_inbox, reply
- [ ] Direct tools: web_search, calculate
- [ ] Decision logic: quick reply → dispatch → continue → cross-reference → close → evolve
- [ ] Living summary protocol: create/update on significant changes

### M4: Self-Evolution

- [ ] update_knowledge: PR to `.agent/knowledge/*.md`
- [ ] update_channels: PR to `.agent/channels.json`
- [ ] create_tool / modify_tool: PR to `.agent/tools/`
- [ ] update_prompt: PR to `.agent/prompt.md`
- [ ] update_model_config: PR to `.agent/models.json`
- [ ] Governance enforcement: auto-merge (knowledge, channels, tools) vs human review (prompt, models)
- [ ] Evolution reasoning: every PR includes why the change is proposed

### M5: Department — Coding Agent

- [ ] Spawn claude-code as subprocess (`--output-format stream-json`)
- [ ] Spawn codex as subprocess (`--json`)
- [ ] Progress streaming: milestone summaries → issue comments
- [ ] Artifact pipeline: diffs inline, large logs → gists
- [ ] PR creation linked to issue
- [ ] Session tracking in DB for potential resume
- [ ] Completion/blocker → notify_inbox
- [ ] Bottom-up handling: department channels route directly to department

### M6: Department — Research

- [ ] Web search + synthesis
- [ ] Results as issue comments
- [ ] Auto-close on completion
- [ ] Bottom-up: direct interaction in department channels

### M7: Cron + Self-Reflection

- [ ] GitHub Actions for hourly/daily triggers
- [ ] Per-task reflection (on issue close): did I have the right tools? what did I learn?
- [ ] Weekly meta-reflection: patterns, missing tools, prompt improvements
- [ ] Journal entries → #journal channel + standing journal issue

### M8: Multi-Model

- [ ] LiteLLM Router integration
- [ ] Model tiers from `.agent/models.json`: fast, smart, coding
- [ ] Fallback chains with error classification (PicoClaw pattern)
- [ ] Cooldown tracking (IronClaw pattern)
- [ ] Cost tracking per interaction

### M9: Semantic Memory

- [ ] Embedding generation for issue content
- [ ] Semantic search tool for manager and departments
- [ ] Memory decay: recent content weighted higher
- [ ] Mem0 integration as optional backend

## Non-Goals for v0

- Slack/CLI adapters (Discord + GitHub only)
- Voice channels
- Browser automation
- WASM sandboxing
- RAG over large document corpora
- Multiple concurrent department instances

## Tech Stack

- **Language**: Python 3.11+ (asyncio)
- **Discord**: discord.py
- **GitHub API**: PyGitHub or `gh` CLI
- **Database**: SQLite + sqlite-vec (search index only)
- **LLM**: Claude API (v0), LiteLLM Router (M8)
- **Coding agents**: Claude Code CLI, Codex CLI (subprocess)
- **Embedding**: text-embedding-3-small or local
- **Cron**: GitHub Actions

## Borrowed Patterns

| Pattern | Source | Milestone |
|---|---|---|
| Channel registry (factory) | NanoClaw | M2 |
| CLI agent spawning (subprocess) | PicoClaw | M5 |
| Asymmetric error recovery | NanoClaw | M2, M5 |
| LiteLLM + retry wrapper | Nanobot | M8 |
| Fallback chain + error classification | PicoClaw | M8 |
| Cooldown-aware failover | IronClaw | M8 |
| Cost tracking + budget guard | IronClaw | M8 |
