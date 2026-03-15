# NaviMem API Reference

Base URL: `https://i.ariseos.com/api/v1/memory`

## Authentication

Three access modes:

**Anonymous** (no headers) — recommended for getting started:
- `/plan`, `/query`: public memory only
- `/learn`: `browser_workflow` type only, writes to public memory
- Rate limit: 30 requests/minute

**API Key** (recommended for production):
```
x-user-id: <user_id>
x-api-key: <api_key>
```
- Full access to private + public memory
- Rate limit: 60 requests/minute

**JWT Bearer Token**:
```
Authorization: Bearer <access_token>
```

---

## POST /api/v1/memory/plan

Memory-powered task planning. Searches memory for relevant workflows and generates a structured execution plan.

**Auth**: Optional. Anonymous → public memory only. Authenticated → private + public.

**Request:**
```json
{"task": "Help me buy a laptop under $500 on Amazon"}
```

**Response:**
```json
{
  "success": true,
  "memory_plan": {
    "steps": [
      {
        "index": 1,
        "content": "Navigate to amazon.com",
        "source": "phrase",
        "phrase_id": "phrase-uuid",
        "state_ids": []
      },
      {
        "index": 2,
        "content": "Search for laptops in the search bar",
        "source": "graph",
        "phrase_id": null,
        "state_ids": ["state-1"]
      }
    ],
    "preferences": [
      "User prefers sorting by customer reviews"
    ],
    "context_hints": [
      "User's budget is under $500"
    ]
  },
  "debug_trace": {
    "agent_messages_count": 6,
    "recalled_phrases": 2,
    "recalled_facts": 3
  }
}
```

**PlanStep fields:**

| Field | Type | Description |
|-------|------|-------------|
| `index` | int | Sequential step number |
| `content` | string | Actionable instruction text |
| `source` | `"phrase"` / `"graph"` / `"none"` | What backs this step |
| `phrase_id` | string? | CognitivePhrase ID when `source="phrase"` |
| `state_ids` | string[] | State IDs when `source="graph"` |

**MemoryPlan fields:**

| Field | Type | Description |
|-------|------|-------------|
| `steps` | PlanStep[] | Ordered execution steps |
| `preferences` | string[] | User preferences from past interactions |
| `context_hints` | string[] | Relevant facts from past conversations |

---

## POST /api/v1/memory/query

Query memory for navigation paths or available actions on a page.

**Auth**: Optional. Anonymous → public memory only. Authenticated → private + public.

**Request:**
```json
{
  "target": "description of target",
  "as_type": "navigation",
  "current_state": null,
  "start_state": "https://example.com/page-a",
  "end_state": "https://example.com/page-b",
  "top_k": 10
}
```

| Field | Type | Description |
|-------|------|-------------|
| `target` | string | Natural language goal/target |
| `as_type` | string? | `"navigation"` or `"action"` |
| `current_state` | string? | Current page URL (for action queries) |
| `start_state` | string? | Starting URL (for navigation queries) |
| `end_state` | string? | Ending URL (for navigation queries) |
| `top_k` | int | Result count (1-100, default 10) |

### Response: navigation query

Returns an ordered path of States + Actions between two locations.

```json
{
  "success": true,
  "query_type": "navigation",
  "source": "public",
  "states": [...],
  "actions": [...]
}
```

### Response: action query

Returns available operations and navigation options from the current page.

```json
{
  "success": true,
  "query_type": "action",
  "source": "public",
  "intent_sequences": [
    {"id": "seq-1", "description": "Login with credentials", "intents": [...]}
  ],
  "outgoing_actions": [
    {"id": "action-1", "source": "state-1", "target": "state-2", "type": "navigate", "description": "Go to dashboard"}
  ]
}
```

---

## POST /api/v1/memory/learn

Learn from execution traces. Converts browser workflow traces into reusable knowledge patterns.

**Auth**: Optional. Anonymous → `browser_workflow` type only, writes to public memory. Authenticated → all types, private + auto-share to public.

**Request:**
```json
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
  ],
  "source": "web-agent"
}
```

**Request fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | `"browser_workflow"` (anonymous) or `"conversation"` / `"mixed"` (authenticated only) |
| `task` | string | Yes | User's original request (natural language) |
| `success` | bool | No | Whether the task succeeded (default: true) |
| `steps` | TraceStep[] | Yes | Browser action sequence |
| `source` | string | No | Client identifier (e.g. `"web-agent"`, `"arise-browser"`) |

**TraceStep fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | string | Yes | Current page URL |
| `action` | string | Yes | `navigate` / `click` / `type` / `scroll` / `select` / `submit` / `done` |
| `target` | string | No | Element description (for click/type/select) |
| `value` | string | No | Input value (for type/select) |
| `thinking` | string | No | Agent's reasoning before this step |
| `success` | bool | No | Whether this step succeeded (null = assume true) |
| `result_summary` | string | No | Compressed result of the step |
| `judgment` | string | No | Agent's assessment after this step |

**Response:**
```json
{
  "success": true,
  "phrase_created": true,
  "phrase_id": "phrase-uuid",
  "phrase_ids": ["phrase-uuid"],
  "shared_phrase_ids": ["public-uuid"],
  "reason": "Workflow captures Amazon search pattern",
  "task_solved": true,
  "execution_clean": true,
  "graph_summary": {
    "domain_count": 1,
    "state_count": 2,
    "page_instance_count": 2,
    "intent_sequence_count": 3,
    "action_count": 1,
    "manage_count": 1,
    "new_states": 2,
    "reused_states": 0,
    "processing_time_ms": 150
  },
  "processing_time_ms": 2500
}
```

**Response fields:**

| Field | Type | Description |
|-------|------|-------------|
| `phrase_created` | bool | Whether a reusable workflow pattern was created |
| `phrase_id` | string? | Created pattern ID |
| `phrase_ids` | string[] | All created pattern IDs |
| `shared_phrase_ids` | string[] | IDs shared to public memory |
| `reason` | string? | Why this workflow was worth learning |
| `task_solved` | bool | Whether the task was actually solved |
| `execution_clean` | bool | Whether execution had no detours or errors |
| `graph_summary` | object | Counts of graph nodes created |

---

## Backward Compatibility

`POST /api/v1/learn-from-trace` is an alias for `/api/v1/memory/learn` with identical behavior.

---

## Endpoint Summary

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/memory/plan` | Optional | Task planning from memory |
| POST | `/memory/query` | Optional | Navigation/action lookup |
| POST | `/memory/learn` | Optional | Report execution trace |
| POST | `/learn-from-trace` | Optional | Alias for `/memory/learn` |
