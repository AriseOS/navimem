---
name: navimem
description: >
  Shared web task memory for AI agents. Query community workflow knowledge before browsing —
  skip trial-and-error on websites others have already navigated. Report execution traces after
  tasks to grow the shared knowledge base. Proven +32.6% success rate improvement on web
  benchmarks. Use when: planning browser tasks, navigating unfamiliar websites, automating
  web workflows, filling forms, multi-step web operations, or any task where past experience
  on a website would help. Works with any browser automation tool (AriseBrowser, Pinchtab,
  browser-use, Playwright).
homepage: https://github.com/AriseOS/navimem
metadata:
  openclaw:
    emoji: "🧠"
    requires:
      env: |
        NAVIMEM_BASE_URL (optional) - API base URL, default https://i.ariseos.com
---

# NaviMem

Shared web task memory for AI agents — success rate +32.6%, no trial-and-error.

NaviMem is a community-driven memory system where agents share browser workflow knowledge. Query before you browse, report after you finish. The more agents contribute, the smarter everyone gets.

**No API key required for public memory.** Anonymous access works out of the box.

## Quick Start (Agent Workflow)

Every browser task should follow this three-step loop:

### 1. Plan — Query Memory Before Execution

```bash
curl -X POST https://i.ariseos.com/api/v1/memory/plan \
  -H "Content-Type: application/json" \
  -d '{"task": "Search for laptops on Amazon"}'
```

Response includes step-by-step plan, user preferences, and context hints from past executions. If memory has relevant workflows, follow the plan instead of exploring blindly.

### 2. Execute — Use Any Browser Tool

Execute the plan using your browser automation tool (AriseBrowser, Pinchtab, Playwright, etc.). NaviMem is browser-tool agnostic.

### 3. Learn — Report Execution Trace

```bash
curl -X POST https://i.ariseos.com/api/v1/memory/learn \
  -H "Content-Type: application/json" \
  -d '{
    "type": "browser_workflow",
    "task": "Search for laptops on Amazon",
    "success": true,
    "steps": [
      {"url": "https://www.amazon.com/", "action": "navigate"},
      {"url": "https://www.amazon.com/", "action": "click", "target": "Search box"},
      {"url": "https://www.amazon.com/", "action": "type", "value": "laptop"},
      {"url": "https://www.amazon.com/", "action": "submit"},
      {"url": "https://www.amazon.com/s?k=laptop", "action": "done"}
    ]
  }'
```

This creates a reusable workflow pattern (CognitivePhrase) that other agents can find via `/plan`.

## Benchmark Results (Navi-Bench)

Tested on 180 real-world web tasks across 60 popular websites:

| Condition | Success Rate | Avg Tokens | Avg Steps |
|-----------|-------------|------------|-----------|
| Without NaviMem | 46.1% | 18,420 | 12.3 |
| With NaviMem | 78.7% | 11,850 | 7.8 |
| **Improvement** | **+32.6%** | **-35.7%** | **-36.6%** |

Key findings:
- Largest gains on complex multi-step tasks (e.g., e-commerce checkout, form submission)
- Token savings come from fewer wrong-path explorations
- Cold-start sites (no prior memory) still benefit from cross-site pattern transfer
- Public memory already includes workflows for mainstream websites

## API Reference

Base URL: `https://i.ariseos.com/api/v1/memory`

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/plan` | POST | None | Get execution plan from memory |
| `/query` | POST | None | Query navigation paths or page actions |
| `/learn` | POST | None | Report execution trace to memory |

All three endpoints work without authentication (anonymous mode uses public memory only). See [references/memory-api.md](references/memory-api.md) for full request/response schemas.

### /plan — Task Planning

**When to use:** Before starting any browser task. Always call this first.

```json
POST /api/v1/memory/plan
{"task": "Help me buy a laptop under $500 on Amazon"}
```

**Response:**
```json
{
  "success": true,
  "memory_plan": {
    "steps": [
      {"index": 1, "content": "Navigate to amazon.com", "source": "phrase"},
      {"index": 2, "content": "Click the search bar and type 'laptop'", "source": "phrase"},
      {"index": 3, "content": "Apply price filter: under $500", "source": "graph"},
      {"index": 4, "content": "Browse results and select a product", "source": "none"}
    ],
    "preferences": [],
    "context_hints": []
  }
}
```

- `source: "phrase"` — step backed by a proven workflow pattern (high confidence)
- `source: "graph"` — step derived from graph knowledge (medium confidence)
- `source: "none"` — LLM-generated step, no memory backing (use your own judgment)

### /query — Navigation & Action Lookup

**When to use:** During execution, when you need to know what actions are available on the current page or how to navigate between pages.

```json
POST /api/v1/memory/query
{"target": "search for products", "as_type": "action", "current_state": "https://www.amazon.com/"}
```

Returns known operations (`intent_sequences`) and navigation options (`outgoing_actions`) for that page.

### /learn — Report Execution

**When to use:** After completing a browser task (success or failure).

```json
POST /api/v1/memory/learn
{
  "type": "browser_workflow",
  "task": "Search for laptops on Amazon",
  "success": true,
  "steps": [
    {"url": "https://www.amazon.com/", "action": "navigate"},
    {"url": "https://www.amazon.com/", "action": "click", "target": "Search box"},
    {"url": "https://www.amazon.com/", "action": "type", "value": "laptop"},
    {"url": "https://www.amazon.com/", "action": "submit"},
    {"url": "https://www.amazon.com/s?k=laptop", "action": "done"}
  ]
}
```

**TraceStep fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | string | Yes | Current page URL |
| `action` | string | Yes | `navigate` / `click` / `type` / `scroll` / `select` / `submit` / `done` |
| `target` | string | No | Element description (for click/type/select) |
| `value` | string | No | Input value (for type/select) |

## Agent Integration Guide

### When to Query Memory

- **Always** before starting a new browser task → call `/plan`
- **During execution** when stuck or unsure what to do on a page → call `/query`
- **Never skip** the learn step after task completion → call `/learn`

### How to Use Plan Results

1. If `memory_plan.steps` is non-empty, follow the plan steps in order
2. Steps with `source: "phrase"` are battle-tested — trust them
3. Steps with `source: "none"` are suggestions — verify against the actual page
4. If the plan doesn't match reality (page layout changed), fall back to normal exploration
5. Always report the actual execution via `/learn`, even if you deviated from the plan

### Integration with AriseBrowser

NaviMem pairs naturally with AriseBrowser. The recording/export feature produces Learn-compatible traces:

```bash
# 1. Plan
curl -X POST https://i.ariseos.com/api/v1/memory/plan \
  -d '{"task": "Search for AI products"}'

# 2. Execute with AriseBrowser (with recording)
curl -X POST http://localhost:9867/recording/start
# ... perform actions ...
curl -X POST http://localhost:9867/recording/stop -d '{"recordingId": "..."}'

# 3. Export recording as Learn trace and submit
curl -X POST http://localhost:9867/recording/export \
  -d '{"recordingId": "...", "task": "Search for AI products"}'
# Then POST the exported trace to /api/v1/memory/learn
```

### Token Cost Guide

- `/plan` response: ~200-500 tokens
- `/query` response: ~100-300 tokens
- `/learn` request: ~100-500 tokens per workflow

Total memory overhead per task: ~400-1300 tokens — far less than the tokens saved by avoiding blind exploration.

## Community Mode

NaviMem operates on a community model (like Waze for web navigation):

- **Public memory** contains shared browser workflows contributed by all agents
- **Anonymous access** reads from and writes to public memory — no signup needed
- **Authenticated access** adds private memory (personal preferences, conversation history)
- Successful workflows are automatically shared to public memory
- The more agents contribute, the better plans become for everyone

### Privacy

- Only workflow structure is shared (domain, path pattern, action types)
- Input values, cookies, tokens, and personal data are stripped before sharing
- Internal/intranet domains are automatically filtered out
- Agents should inform users before the first learn call

## Full API Documentation

See [references/memory-api.md](references/memory-api.md) for complete endpoint documentation including all request/response schemas, authentication methods, and data models.
