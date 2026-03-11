# Spec: Manager Agent

## Role

The manager is the single persistent brain. It is triggered by GitHub Issue events. Its input is always a GitHub Issue with comments. It never sees raw channel messages — the concierge has already recorded them on an issue.

## Input

A GitHub Issue with a new comment:

```python
ManagerInput(
    issue_number=72,
    issue_title="btc price check",
    issue_labels=["incoming"],
    latest_comment="**[discord/#general] @jun**: what's btc price",
    all_comments=[...],  # last N comments
)
```

## Context Loading

Before every decision, the manager loads:

```python
context = {
    # This issue (the immediate context)
    "issue":       gh.get_issue_with_comments(issue_number, last=20),

    # Global state (what's happening across the system)
    "board":       gh.get_project_board_summary(),
    "active":      gh.issues(state="open", label="in-progress"),
    "blocked":     gh.issues(state="open", label="needs-attention"),
    "open_prs":    gh.pulls(state="open"),

    # Related context (what might be relevant)
    "related":     gh.search_issues(extract_keywords(latest_comment)),

    # System config
    "channels":    read(".agent/channels.json"),
    "knowledge":   read_all(".agent/knowledge/*.md"),

    # Search aid (from DB)
    "semantic":    db.search(latest_comment, limit=5),
}
```

## Tool Categories

### 1. Office Tools (GitHub state management)

These are the core tools for managing the source of truth.

```python
# Task lifecycle
create_issue(title: str, body: str, labels: list[str]) -> int
update_issue(number: int, title: str = None, body: str = None, state: str = None)
close_issue(number: int, summary: str)

# Communication on issues
add_comment(number: int, body: str)

# State transitions
add_label(number: int, label: str)
remove_label(number: int, label: str)
move_card(number: int, column: str)  # "Backlog" | "In Progress" | "Review" | "Done"
assign_issue(number: int, user: str)
link_issues(from_number: int, to_number: int)

# Artifacts
create_gist(filename: str, content: str) -> str  # returns URL
```

### 2. Department Tools (dispatch work)

```python
spawn_coding_agent(
    issue: int,
    task: str,
    repo: str,
    agent: str = "claude-code",  # or "codex"
) -> str  # returns session ID

spawn_research(
    issue: int,
    query: str,
) -> str
```

### 3. Communication Tools (channel output)

```python
post_to_channel(channel: str, content: str)
notify_inbox(content: str)
reply(content: str)  # replies in the source channel of the current issue
```

### 4. Self-Evolution Tools (system modification)

All self-evolution happens through PRs. The manager proposes changes, governance rules determine if they auto-merge or need review.

```python
# Knowledge (auto-merge)
update_knowledge(
    file: str,          # e.g., "preferences.md"
    content: str,       # new content
    reason: str,        # why this change
)

# Channel map (auto-merge with notification)
update_channels(
    changes: dict,      # channel config changes
    reason: str,
)

# Tools (auto-merge with notification)
create_tool(
    name: str,          # tool name
    code: str,          # Python source
    description: str,   # what it does
    parameters: dict,   # parameter schema
)

modify_tool(
    name: str,
    code: str,
    reason: str,
)

# Prompt (requires human review)
update_prompt(
    diff_description: str,  # what to change and why
    new_content: str,       # proposed new prompt.md
)

# Model config (requires human review)
update_model_config(
    changes: dict,
    reason: str,
)
```

### 5. Direct Tools (for quick answers)

```python
web_search(query: str) -> str
get_current_time() -> str
calculate(expression: str) -> str
```

## Decision Logic

The manager reads the issue and decides what to do. This is the core decision tree:

```
Read issue + comments
    │
    ├── Is this a quick question?
    │   (factual, lookup, status check, no multi-step work)
    │   → use direct tools (web_search, etc.)
    │   → add_comment with answer
    │   → add_label("quick")
    │   → close_issue with summary
    │   → reply in source channel
    │
    ├── Is this a new task?
    │   (multi-step work, needs coding/research/tracking)
    │   → add_label("in-progress" + domain labels)
    │   → move_card("In Progress")
    │   → spawn_coding_agent or spawn_research
    │   → post_to_channel (department channel)
    │   → reply in source channel with confirmation + issue link
    │
    ├── Is this continuing an existing task?
    │   (issue already has in-progress label, comments are ongoing)
    │   → read living summary + recent comments
    │   → continue work or update dispatch
    │   → add_comment with update
    │
    ├── Does this relate to another issue?
    │   (found via search or semantic match)
    │   → link_issues
    │   → add_comment on related issue noting the connection
    │
    ├── Did a department complete work?
    │   (PR merged, research done, task finished)
    │   → close_issue with final summary
    │   → move_card("Done")
    │   → notify_inbox
    │
    ├── Is something blocked?
    │   → add_label("needs-attention")
    │   → notify_inbox
    │
    └── Did I learn something worth remembering?
        (user preference, factual correction, pattern)
        → update_knowledge
```

## Self-Evolution Scenarios

### Learning a preference
```
User: "i prefer non-custodial wallets"
Manager: (answers the question normally)
Manager: update_knowledge("preferences.md",
    append="- prefers non-custodial crypto solutions",
    reason="user stated preference directly")
→ PR to .agent/knowledge/preferences.md → auto-merge
```

### Noticing a missing tool
```
Manager: (just did web_search for crypto prices for the 5th time)
Manager: create_tool(
    name="crypto_price",
    code="import httpx\nasync def crypto_price(tokens)...",
    description="Get live crypto token prices from CoinGecko",
    parameters={"tokens": {"type": "array"}},
)
→ PR adding .agent/tools/crypto_price.py → auto-merge with notification
```

### Proposing a new channel
```
Manager: (user keeps asking crypto questions in #general)
Manager: update_channels(
    changes={"discord/#crypto": {"type": "department", "department": "research", "default_labels": ["crypto"]}},
    reason="user frequently asks crypto questions, dedicated channel would improve organization",
)
→ PR to .agent/channels.json → auto-merge with notification
→ notify_inbox("Proposed new #crypto channel — check PR")
```

### Refining own prompt
```
Manager: (weekly reflection notices it's been too verbose)
Manager: update_prompt(
    diff_description="Add principle: be concise, lead with the answer",
    new_content="...",
)
→ PR to .agent/prompt.md → requires human review
→ notify_inbox("Proposed prompt update — needs your review")
```

### Changing model preference
```
Manager: (notices haiku is failing on certain crypto queries)
Manager: update_model_config(
    changes={"tiers.fast.models": ["gemini/gemini-2.5-flash"]},
    reason="haiku struggles with real-time price queries, gemini-flash handles them better",
)
→ PR to .agent/models.json → requires human review
```

## Living Summary Protocol

Every issue gets a living summary as the first comment (or issue body), updated on significant changes:

```markdown
## Status
in-progress | needs-review | blocked | done

## Summary
One-line description of what this is about.

## Key Decisions
- Decided X because Y (Mar 10)
- Changed approach to Z (Mar 11)

## Progress
- [x] Initial research
- [x] Compared 3 options
- [ ] Final recommendation

## Open Questions
- What's the risk tolerance?

## Related
- #65 (SSO feature)
- #42 (portfolio strategy)

## Sessions
- cc_abc123: initial research (Mar 10)
- cc_def456: follow-up (Mar 11)
```

## Example Flows

### "what's btc price" (quick, seconds)

```
Concierge: create Issue #80 "btc price check"
  → comment: [discord/#general] @jun: what's btc price

Manager reads #80:
  → web_search("bitcoin price current")
  → add_comment(#80, "$67,420, up 3.2% today")
  → add_label(#80, "quick")
  → close_issue(#80, "BTC price: $67,420 (+3.2%)")
  → reply("$67,420, up 3.2% today")
```

### "change footer to XYZ" (task, hours)

```
Concierge: create Issue #81 "change footer text"
  → comment: [discord/#general] @jun: change footer text from ABC to XYZ

Manager reads #81:
  → add_label(#81, "in-progress", "project-x")
  → move_card(#81, "In Progress")
  → spawn_coding_agent(#81, "change footer text from ABC to XYZ", repo="junyaoc/project-x")
  → post_to_channel("discord/#project-x", "starting footer change — #81")
  → reply("on it — tracking in #project-x as #81")

(claude-code works, posts progress as comments on #81)

Manager reads #81 (on completion):
  → add_comment(#81, "Footer changed, PR #82 ready")
  → add_label(#81, "needs-review")
  → move_card(#81, "Review")
  → notify_inbox("footer change done — PR #82 needs review")
```

### User in #project-x: "auth broken" (bottom-up)

```
Concierge: create Issue #83 "auth broken on safari"
  → labels: ["incoming", "project-x"]
  → comment: [discord/#project-x] @jun: auth broken on safari

Department handler (channel type = department):
  → add_label(#83, "bug", "in-progress")
  → investigates via coding agent
  → add_comment(#83, progress updates)
  → opens PR #84
  → add_label(#83, "needs-review")
  → notify_inbox("safari auth fix — PR #84")
```

### User on GitHub directly

```
User comments on Issue #81: "also make it bold"
  → webhook fires
  → manager_handler(issue=#81)
  → Manager reads #81 + new comment
  → continues work, updates department
  → post_to_channel("discord/#project-x", "new requirement on #81")
```
