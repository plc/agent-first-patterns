# Agent-First API Design Patterns

A reference of design patterns for building APIs that AI agents can discover, learn, and use autonomously. Each pattern includes the problem it solves, how to implement it, and concrete examples from [CalDave](https://caldave.ai), a production calendar-as-a-service API for AI agents.

CalDave is used as the reference implementation throughout this document. All code examples, API calls, and response shapes are real production code from the CalDave codebase.

---

## Table of Contents

1. [Self-Describing API Manual](#1-self-describing-api-manual)
2. [Progressive Recommendation Engine](#2-progressive-recommendation-engine)
3. [Soft Authentication](#3-soft-authentication)
4. [Zero-Auth Agent Bootstrapping](#4-zero-auth-agent-bootstrapping)
5. [Context Window Management](#5-context-window-management)
6. [Generated Examples with Real Credentials](#6-generated-examples-with-real-credentials)
7. [Agent-Aware Changelog](#7-agent-aware-changelog)
8. [Unknown Field Rejection](#8-unknown-field-rejection)
9. [Type-Prefixed IDs](#9-type-prefixed-ids)
10. [Consistent Error Format](#10-consistent-error-format)
11. [404 as Discovery Mechanism](#11-404-as-discovery-mechanism)
12. [Agent-Scoped Error Introspection](#12-agent-scoped-error-introspection)
13. [Polling Hints](#13-polling-hints)
14. [Webhook System](#14-webhook-system)
15. [Field Alias Acceptance](#15-field-alias-acceptance)
16. [Arbitrary Metadata Field](#16-arbitrary-metadata-field)
17. [Plain Text Rendering](#17-plain-text-rendering)
18. [Protocol Abstraction](#18-protocol-abstraction)
19. [Progressive Enhancement](#19-progressive-enhancement)
20. [Agent-First Ownership Model](#20-agent-first-ownership-model)
21. [Rate Limiting with Standard Headers](#21-rate-limiting-with-standard-headers)
22. [MCP Support](#22-mcp-support)
23. [Seed Data on Resource Creation](#23-seed-data-on-resource-creation)
24. [Pre-Computed URLs in Responses](#24-pre-computed-urls-in-responses)
25. [Fire-and-Forget Webhook Delivery](#25-fire-and-forget-webhook-delivery)

---

## 1. Self-Describing API Manual

**Problem:** AI agents have no prior knowledge of your API. Traditional docs are HTML pages designed for human eyes. An agent needs to discover capabilities, authenticate, and start making calls -- all programmatically.

**Pattern:** Expose a single endpoint that returns the complete API reference as structured JSON. An agent can go from zero knowledge to full API usage with one HTTP call.

### CalDave example

CalDave exposes `GET /man` (short for "manual") that returns a complete JSON description of every endpoint. The endpoint catalog is pre-computed at module load time in src/routes/man.js for performance. Each endpoint entry includes method, path, description, auth requirement, complete parameter list, and example request/response. An agent calling `GET /man` receives everything it needs to start using the API.

```bash
# Unauthenticated -- discover the full API
curl -s https://caldave.ai/man | jq '.endpoints | length'
# => 24

# Authenticated -- get personalized context and recommendations
curl -s https://caldave.ai/man \
  -H "Authorization: Bearer sk_live_..."
```

The response includes:
- `overview` -- what CalDave does
- `base_url` -- https://caldave.ai
- `available_topics` -- ["agents", "calendars", "events", "feeds", "smtp", "errors"]
- `rate_limits` -- enforcement rules and headers
- `error_format` -- the consistent error shape used throughout the API
- `your_context` -- the agent's current state (calendars, event counts, claimed status)
- `recommended_next_step` -- personalized action with ready-to-run curl command
- `endpoints` -- complete catalog with generated examples

---

## 2. Progressive Recommendation Engine

**Problem:** An agent that reads 30 endpoints doesn't know which one to call first. Without guidance, agents make random or incorrect choices about what to do next.

**Pattern:** Inspect the agent's current state and return the single most important next action with a ready-to-execute curl command. The recommendation changes as the agent progresses through setup.

### CalDave example

CalDave's `GET /man` response includes a `recommended_next_step` field. The server inspects the agent's current state in src/routes/man.js -- whether it's authenticated, whether it has a name, calendars, events, and whether it's claimed by a human -- and returns the single most important next action. Each recommendation includes an action description, the endpoint to call, and a ready-to-execute curl command with the agent's real credentials.

The recommendation chain progresses through these steps:

1. Not authenticated -> "Create an agent" (POST /agents)
2. No name -> "Name your agent" (PATCH /agents)
3. No calendars -> "Create your first calendar" (POST /calendars)
4. No events -> "Create your first event" (POST /calendars/:id/events)
5. Not claimed -> "Claim this agent with a human account" (POST /agents/claim)
6. Everything set up -> "Check upcoming events" (GET /calendars/:id/upcoming)

Each step provides an executable curl command with the agent's real credentials interpolated.

---

## 3. Soft Authentication

**Problem:** Discovery endpoints need to work without authentication (so new agents can find out how to create credentials), but authenticated agents should get personalized data. Standard auth middleware returns 401 for missing tokens, blocking unauthenticated access entirely.

**Pattern:** "Soft auth" resolves a Bearer token when present but never returns 401. If the token is valid, `req.agent` is populated. If missing or invalid, `req.agent` is set to `null` and the request proceeds.

### CalDave example

CalDave implements soft auth for discovery endpoints (`/man`, `/changelog`) in src/routes/man.js. The middleware checks for an Authorization header, and if present and valid, populates `req.agent` with the agent's data. If the header is missing or invalid, `req.agent` is set to null and the request continues without authentication. Database errors also don't block the request -- they just result in null agent context.

Compare with hard auth used on resource endpoints in src/middleware/auth.js, which returns 401 if the Authorization header is missing or invalid.

CalDave uses three auth styles:

| Endpoint type | Auth style | Example |
|---|---|---|
| Discovery | Soft auth (never 401) | `GET /man`, `GET /changelog` |
| Resource CRUD | Hard auth (401 if missing) | `POST /calendars/:id/events` |
| Agent creation | No auth | `POST /agents` |
| Webhooks/feeds | Token in URL | `GET /feeds/:id.ics?token=feed_...` |

---

## 4. Zero-Auth Agent Bootstrapping

**Problem:** Traditional APIs require a human to create an account, log in, navigate to a dashboard, generate an API key, and configure it. AI agents can't do any of that.

**Pattern:** A single unauthenticated POST creates an agent identity and returns a usable API key. The agent goes from zero credentials to fully authenticated in one HTTP call.

### CalDave example

CalDave's `POST /agents` endpoint requires no authentication. The implementation in src/routes/agents.js generates a unique agent ID (agt_ prefix), creates a random API key (sk_live_ prefix), hashes it with SHA-256, and stores only the hash in the database. The endpoint validates optional name and description fields, inserts the agent record, and returns the agent_id and api_key in the response with a warning that the key will only be shown once.

An agent can bootstrap in seconds:

```bash
# 1. Create agent (no auth required)
RESPONSE=$(curl -s -X POST https://caldave.ai/agents \
  -H "Content-Type: application/json" \
  -d '{"name": "My Calendar Agent"}')
API_KEY=$(echo $RESPONSE | jq -r '.api_key')

# 2. Create a calendar
curl -s -X POST https://caldave.ai/calendars \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Work", "timezone": "America/New_York"}'

# 3. Start using it
curl -s https://caldave.ai/calendars \
  -H "Authorization: Bearer $API_KEY"
```

Key details:
- The API key is shown exactly once
- Only the SHA-256 hash is stored (not the plaintext key)
- `name` and `description` are optional
- Rate-limited to 20 requests/hour per IP to prevent abuse
- Optional `X-Human-Key` header allows auto-association with a human account

---

## 5. Context Window Management

**Problem:** AI agents have limited context windows. The full `/man` response with 30+ endpoints can be thousands of tokens. An agent working on one feature doesn't need to see endpoints for unrelated features.

**Pattern:** Two mechanisms to reduce payload size:

1. **Topic filtering** (`?topic=`) -- returns only endpoints matching specified categories
2. **Guide mode** (`?guide`) -- returns only overview and recommendation, no endpoint catalog

### CalDave example

CalDave supports topic filtering on `GET /man` in src/routes/man.js. The server validates the requested topics against a whitelist (agents, calendars, events, feeds, smtp, errors), filters the endpoint list to only include matching topics plus discovery endpoints (which are always included), and returns helpful errors if invalid topics are requested.

Usage:

```bash
# Full catalog -- 24 endpoints
curl -s https://caldave.ai/man -H "Authorization: Bearer sk_live_..." | jq '.endpoints | length'
# => 24

# Only event-related endpoints
curl -s "https://caldave.ai/man?topic=events" -H "Authorization: Bearer sk_live_..." | jq '.endpoints | length'
# => 8 (events endpoints + discovery endpoints)

# Multiple topics
curl -s "https://caldave.ai/man?topic=events,calendars" -H "Authorization: Bearer sk_live_..."

# Invalid topic returns helpful error
curl -s "https://caldave.ai/man?topic=bogus"
# => { "error": "Unknown topic: bogus. Available: agents, calendars, events, feeds, smtp, errors" }
```

Guide mode returns only context and recommendation. When the `?guide` query parameter is present, the response includes overview, base_url, rate_limits, error_format, your_context, recommended_next_step, and a discover_more object with links to the full API reference and other resources. The endpoint catalog is completely omitted, significantly reducing token usage.

```bash
# Guide mode -- much smaller response
curl -s "https://caldave.ai/man?guide" -H "Authorization: Bearer sk_live_..."
# Returns only overview, context, and next step recommendation -- no endpoint catalog
```

---

## 6. Generated Examples with Real Credentials

**Problem:** Example code in docs typically uses placeholder values like `YOUR_API_KEY` and `YOUR_CALENDAR_ID`. An agent has to figure out what to substitute. This is error-prone and wastes tokens.

**Pattern:** When the agent is authenticated, every curl example in the `/man` response is generated with the agent's actual API key and real resource IDs interpolated.

### CalDave example

CalDave's `buildCurl()` function in src/routes/man.js generates curl commands with real credentials. The function takes a method, path, and optional parameters (apiKey, calId, evtId, body, queryString), resolves path parameters by replacing placeholders like :id with actual resource IDs, and constructs a properly formatted curl command with Authorization headers and JSON body when needed. If no credentials are provided, it falls back to placeholder values.

When unauthenticated, examples use placeholders:

```bash
curl -s -X POST "https://caldave.ai/calendars/CAL_ID/events" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title":"Team standup","start":"2026-04-10T10:00:00Z","end":"2026-04-10T10:30:00Z"}'
```

When authenticated, examples use real values from the agent's account:

```bash
curl -s -X POST "https://caldave.ai/calendars/cal_a1b2c3XyZw/events" \
  -H "Authorization: Bearer sk_live_x7y8z9AbCdEfGh" \
  -H "Content-Type: application/json" \
  -d '{"title":"Team standup","start":"2026-04-10T10:00:00Z","end":"2026-04-10T10:30:00Z"}'
```

The agent can copy any curl command from the `/man` response and execute it immediately without substitution. This dramatically reduces friction during API exploration.

---

## 7. Agent-Aware Changelog

**Problem:** Agents don't know when new features are added. They continue using the same endpoints they were originally programmed with, missing improvements.

**Pattern:** A structured changelog endpoint that, when authenticated, splits entries into "changes since your agent was created" vs "changes before you existed." Includes personalized recommendations and explicit polling guidance.

### CalDave example

CalDave's `GET /changelog` endpoint uses soft auth in src/routes/changelog.js to provide personalized change tracking. The endpoint maintains a structured CHANGELOG array with entries that include date, type (feature, improvement, fix, breaking), title, description, affected endpoints, and documentation links. When authenticated, it fetches the agent's created_at timestamp, filters the changelog to find entries newer than that date, builds personalized recommendations based on the agent's current setup, and returns both the full changelog and the filtered "changes since signup" list.

Authenticated response shows what's new:

```bash
curl -s https://caldave.ai/changelog \
  -H "Authorization: Bearer sk_live_..."
```

```json
{
  "description": "API changelog. Lists new features, improvements, and fixes.",
  "poll_recommendation": "Check this endpoint approximately once per week.",
  "total_changes": 12,
  "your_agent": {
    "agent_id": "agt_abc123",
    "name": "My Calendar Agent",
    "created_at": "2026-03-15T10:00:00.000Z"
  },
  "changes_since_signup": [
    {
      "date": "2026-03-31",
      "changes": [
        {
          "type": "feature",
          "title": "Time-based webhooks",
          "description": "...",
          "endpoints": ["POST /calendars"],
          "docs": "https://caldave.ai/docs#webhooks"
        }
      ]
    }
  ],
  "changes_since_signup_count": 3,
  "changelog": [ "...all entries..." ],
  "recommendations": [
    {
      "action": "Set up webhooks on your calendar",
      "why": "You have a calendar but no webhooks configured. Webhooks let you react to events in real-time.",
      "how": "PATCH /calendars/:id with {\"webhook_url\": \"https://...\", \"webhook_secret\": \"...\"}",
      "docs": "https://caldave.ai/docs#webhooks"
    }
  ]
}
```

Each changelog entry has a `type` field (`feature`, `improvement`, `fix`, `breaking`) so agents can programmatically filter for what matters.

---

## 8. Unknown Field Rejection

**Problem:** LLMs frequently hallucinate field names. If `due_date` is the internal column name but the API accepts `due`, an agent might send `due_date` and the API silently ignores it. The agent thinks it set the due date, but nothing happened.

**Pattern:** Every write endpoint validates incoming fields against a known set and returns a specific error listing the unknown fields. No field is ever silently ignored.

### CalDave example

CalDave defines a Set of known fields for every resource type (events, calendars, agents) in their respective route files. On every write endpoint, incoming fields are checked against this set in a checkUnknownFields() helper function before processing. Unknown fields trigger a 400 error listing exactly which fields were unrecognized. This is applied consistently across all write endpoints in the API. The pattern appears in src/routes/events.js, src/routes/agents.js, and src/routes/calendars.js.

Example of the error an agent receives:

```bash
curl -s -X POST https://caldave.ai/calendars/cal_abc/events \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{"title": "Test", "start_time": "2026-04-01T10:00:00Z", "end_time": "2026-04-01T11:00:00Z"}'
```

Response (400):

```json
{
  "error": "Unknown fields: start_time, end_time"
}
```

The agent immediately knows `start_time` and `end_time` aren't valid. It can check `/man` to find the correct field names (`start` and `end`).

CalDave applies this pattern consistently across all resource types with separate Sets for create vs patch operations where needed.

---

## 9. Type-Prefixed IDs

**Problem:** APIs that use UUIDs or numeric IDs make it easy to pass a calendar ID where an event ID is expected. The error message ("not found") doesn't tell the agent it used the wrong type of ID.

**Pattern:** Every ID includes a prefix that identifies its entity type. An agent (or a human reviewing logs) can immediately tell what kind of entity an ID refers to.

### CalDave example

CalDave uses nanoid with type prefixes for all IDs. The implementation in src/lib/ids.js creates ID generators using customAlphabet with a base-62 alphanumeric character set. Short IDs (12 chars) are used for entities like agents, calendars, and events that appear in URLs. Long IDs (32 chars) are used for secrets, tokens, and API keys that need high entropy. Each generator function adds a descriptive prefix: agt_ for agents, cal_ for calendars, evt_ for events, feed_ for feed tokens, inb_ for inbound tokens, sk_live_ for agent API keys, hum_ for humans, ha_ for human-agent associations, sess_ for sessions, and hk_live_ for human API keys.

Design decisions:
- **Short IDs** (12 random chars) for entities that appear in URLs and logs
- **Long IDs** (32 random chars) for secrets, tokens, and API keys
- **Alphanumeric alphabet** -- no special chars, so IDs are URL-safe without encoding
- **Consistent prefix convention** -- agents learn the pattern and can self-check before sending requests

Example response showing type-prefixed IDs:

```json
{
  "agent_id": "agt_x7y8z9AbCdEf",
  "api_key": "sk_live_abc123...",
  "calendars": [
    {
      "calendar_id": "cal_a1b2c3XyZwVu",
      "name": "Work",
      "feed_token": "feed_def456...",
      "inbound_webhook_url": "https://caldave.ai/inbound/inb_ghi789..."
    }
  ]
}
```

If an agent tries to use an event ID where a calendar ID is expected, the error makes it obvious:

```bash
curl -s https://caldave.ai/calendars/evt_p4q5r6StUvWx/events \
  -H "Authorization: Bearer sk_live_..."
# => { "error": "Calendar not found" }
```

The agent (or human debugging) can see `evt_` in the URL and realize the mistake immediately.

---

## 10. Consistent Error Format

**Problem:** Inconsistent error formats force agents to handle multiple shapes. Some APIs return `{ "message": "..." }`, others `{ "errors": [...] }`, others `{ "code": 123, "detail": "..." }`.

**Pattern:** Every error response in the entire API uses the same shape: `{ "error": "Human-readable message" }`. Document this in the `/man` response so agents know what to expect before they encounter errors.

### CalDave example

Every error in CalDave uses `{ "error": "..." }` format across all routes:

- 400 - Validation errors like "name is required", "Unknown fields: start_time, end_time", "title must be a string", "description exceeds 64KB limit"
- 401 - Auth errors like "Missing or invalid Authorization header"
- 404 - Not found errors like "Calendar not found", "Event not found"
- 404 - Global catch-all: "Not found. Try GET /man for the API reference."
- 429 - Rate limit: "Too many requests, please try again later"
- 500 - Server errors like "Failed to create event", "Failed to send invite"

The error format is documented in the ERROR_FORMAT constant in src/routes/man.js so agents know what to expect. The format documentation includes the shape, a description explaining all errors use this consistent format, and a mapping of status codes to their meanings with guidance on how to handle each one.

Agent code to handle any CalDave error requires only a single check for the `error` field, providing one consistent error handling path for all errors across the entire API.

---

## 11. 404 as Discovery Mechanism

**Problem:** When an agent hits a wrong URL, it gets a generic "Not Found" and has no idea where to go next.

**Pattern:** The 404 catch-all response points the agent to the API manual.

### CalDave example

CalDave's global 404 handler in src/index.js provides discovery guidance in the error message, directing agents to GET /man to discover valid endpoints.

Every wrong URL becomes a learning opportunity:

```bash
curl -s https://caldave.ai/bogus-endpoint
# => { "error": "Not found. Try GET /man for the API reference." }

curl -s https://caldave.ai/man | jq '.endpoints | length'
# => 24
```

The agent can immediately call `GET /man` to discover the correct endpoints. This pattern turns navigation errors into discovery moments.

---

## 12. Agent-Scoped Error Introspection

**Problem:** When an agent triggers a server error (500), traditional APIs just say "Internal server error." The agent can't debug the issue, and a human has to check server logs.

**Pattern:** Log all server errors to a database table with the agent's ID attached. Expose a `GET /errors` endpoint so agents can query their own errors.

### CalDave example

CalDave logs errors to the `error_log` table using a logError() helper function in src/lib/errors.js. The function extracts the error message and stack trace, then inserts a record with route, method, status code, message, stack, and agent_id. This is called from every catch block throughout the codebase.

Agents can debug their own errors through two endpoints defined in src/index.js. `GET /errors` accepts optional limit and route query parameters, queries the error_log table filtered by the authenticated agent's ID, and returns a list of recent errors with pagination. `GET /errors/:id` retrieves the full details of a specific error including stack trace, also filtered by agent_id for security.

Usage:

```bash
# List recent errors
curl -s https://caldave.ai/errors \
  -H "Authorization: Bearer sk_live_..."

# Response:
{
  "errors": [
    {
      "id": 42,
      "route": "POST /calendars/:id/events",
      "method": "POST",
      "status_code": 500,
      "message": "invalid input syntax for type timestamp: \"not-a-date\"",
      "created_at": "2026-04-01T15:30:00.000Z"
    }
  ],
  "count": 1
}

# Get full stack trace
curl -s https://caldave.ai/errors/42 \
  -H "Authorization: Bearer sk_live_..."

# Filter by route
curl -s "https://caldave.ai/errors?route=events" \
  -H "Authorization: Bearer sk_live_..."
```

Errors are scoped by agent_id -- an agent can only see errors from its own API calls.

---

## 13. Polling Hints

**Problem:** An agent that polls on a fixed interval wastes API calls when nothing is happening soon, and misses things when timing is tight.

**Pattern:** Include machine-readable hints that tell the agent when to poll next.

### CalDave example

CalDave returns ISO 8601 duration hints on time-sensitive endpoints. In src/routes/events.js, the msToIsoDuration() helper converts milliseconds to ISO 8601 duration format (like PT14M30S). The GET /calendars/:id/upcoming endpoint calculates the time until the next event starts, converts it to ISO 8601 duration format, and includes it in the response as next_event_starts_in.

Response with polling hint:

```json
{
  "events": [
    {
      "event_id": "evt_abc123",
      "title": "Team standup",
      "start": "2026-04-02T10:00:00.000Z",
      "end": "2026-04-02T10:30:00.000Z"
    }
  ],
  "next_event_starts_in": "PT14M30S"
}
```

`PT14M30S` means "14 minutes and 30 seconds" (ISO 8601 duration format). The agent schedules its next poll accordingly instead of using a fixed interval. Parseable by standard libraries in most languages.

For low-frequency endpoints like `/changelog`, CalDave provides explicit frequency guidance:

```json
{
  "poll_recommendation": "Check this endpoint approximately once per week.",
  "changelog": [ "..." ]
}
```

---

## 14. Webhook System

**Problem:** Polling wastes resources and introduces latency. Agents need to know immediately when things change.

**Pattern:** Support two categories of webhooks:
1. **Mutation webhooks** -- fire immediately when data changes
2. **Time-based webhooks** -- fire from a background poller when scheduled times arrive

### CalDave example

CalDave supports six webhook types. Mutation webhooks (fired from route handlers) include event.created, event.updated, event.deleted, and event.responded. Time-based webhooks (fired from a background poller) include event.starting (5 minutes before) and event.ending (when event ends).

Setting up webhooks on a calendar:

```bash
curl -s -X POST https://caldave.ai/calendars \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Work",
    "timezone": "America/New_York",
    "webhook_url": "https://my-agent.example.com/webhooks",
    "webhook_secret": "my-hmac-secret-123"
  }'
```

Webhook test endpoint implementation in src/routes/calendars.js retrieves the calendar, verifies it has a webhook_url configured, constructs a test payload, adds HMAC signature if webhook_secret is configured, sends the request with a 10-second timeout, and returns success or failure with detailed status information.

Webhook payload format:

```json
{
  "type": "event.created",
  "calendar_id": "cal_abc123",
  "event_id": "evt_xyz789",
  "event": {
    "event_id": "evt_xyz789",
    "title": "Team standup",
    "start": "2026-04-10T10:00:00.000Z",
    "end": "2026-04-10T10:30:00.000Z",
    "status": "confirmed"
  },
  "timestamp": "2026-04-02T18:30:00.000Z"
}
```

HMAC-SHA256 signing is implemented in src/lib/webhooks.js. If webhook_secret is configured, the server computes a SHA-256 HMAC of the request body and includes it as the X-CalDave-Signature header. Agents should verify this signature by computing the HMAC of the raw body string (not re-serialized JSON) and comparing using timing-safe comparison.

---

## 15. Field Alias Acceptance

**Problem:** LLMs hallucinate field names based on training data. If the more common term in public codebases differs from your API's field name, agents will consistently get it wrong.

**Pattern:** Accept common aliases for fields that agents are likely to confuse. Normalize silently before validation.

### CalDave example

CalDave accepts `rrule` as an alias for `recurrence` in src/routes/events.js. The normalizeBody() function checks if rrule is present and recurrence is not, copies rrule to recurrence, deletes the rrule field, and returns the normalized body. Both fields are included in KNOWN_EVENT_FIELDS so neither triggers unknown field rejection during the validation step.

Usage -- both fields work identically:

```bash
# Using canonical field name
curl -s -X POST https://caldave.ai/calendars/cal_abc/events \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{"title": "Weekly meeting", "start": "2026-04-10T10:00:00Z", "end": "2026-04-10T11:00:00Z", "recurrence": "FREQ=WEEKLY"}'

# Using alias (rrule is more common in iCal contexts)
curl -s -X POST https://caldave.ai/calendars/cal_abc/events \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{"title": "Weekly meeting", "start": "2026-04-10T10:00:00Z", "end": "2026-04-10T11:00:00Z", "rrule": "FREQ=WEEKLY"}'
```

Both requests work identically. The agent doesn't get an "unknown field" error when using the more common term from training data.

---

## 16. Arbitrary Metadata Field

**Problem:** Different agents need to store different auxiliary data on resources. One agent might track an external CRM ID, another might store processing state, another might attach tags. A rigid schema can't anticipate every agent's needs.

**Pattern:** Resources have a `metadata` JSONB field that accepts arbitrary structured data up to a size limit. The API stores it as-is and returns it as-is.

### CalDave example

CalDave events accept a `metadata` field validated in src/routes/events.js. The validation checks that metadata is an object (not an array, null, or primitive type), calculates the size of the serialized JSON, and enforces a 16KB size limit. Otherwise the field is stored as-is with no schema validation.

Usage:

```bash
curl -s -X POST https://caldave.ai/calendars/cal_abc/events \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Follow up with client",
    "start": "2026-04-10T14:00:00Z",
    "end": "2026-04-10T14:30:00Z",
    "metadata": {
      "crm_deal_id": "deal_789",
      "source": "email_triage",
      "priority_score": 0.85,
      "tags": ["client", "urgent"],
      "internal_notes": "Discussed pricing, needs proposal by Friday"
    }
  }'
```

The metadata is returned in all responses for that event:

```json
{
  "event_id": "evt_abc123",
  "title": "Follow up with client",
  "start": "2026-04-10T14:00:00Z",
  "end": "2026-04-10T14:30:00Z",
  "metadata": {
    "crm_deal_id": "deal_789",
    "source": "email_triage",
    "priority_score": 0.85,
    "tags": ["client", "urgent"],
    "internal_notes": "Discussed pricing, needs proposal by Friday"
  }
}
```

CalDave validates that metadata is an object (not an array or primitive) and enforces a 16KB size limit, but otherwise stores it as-is. Different agents can use this field for completely different purposes.

---

## 17. Plain Text Rendering

**Problem:** Some agents operate in text-only environments (terminals, chat interfaces, simple HTTP clients). Parsing JSON to produce a human-readable view adds unnecessary complexity.

**Pattern:** Offer a `text/plain` endpoint that returns pre-formatted output that works directly with `curl`.

### CalDave example

CalDave provides `GET /calendars/:id/view` for plain text output. The implementation in src/routes/view.js verifies calendar ownership, fetches upcoming events ordered by start time, formats them as a fixed-width table with columns for title, start, end, location, and status, and returns the result with Content-Type text/plain. The formatting includes the calendar name, timezone, a separator line, column headers, event rows with padding, and a summary line showing the event count.

Usage:

```bash
curl -s https://caldave.ai/calendars/cal_abc123/view \
  -H "Authorization: Bearer sk_live_..."
```

Response (text/plain):

```
Work (cal_abc123)  tz: America/New_York
--------------------------------------------------------------------------------
TITLE                          START                  END                    LOCATION             STATUS
--------------------------------------------------------------------------------
Team standup                   2026-04-02 10:00:00Z   2026-04-02 10:30:00Z   Zoom                 confirmed
Client review                  2026-04-02 14:00:00Z   2026-04-02 15:00:00Z   Office               confirmed
Sprint planning                2026-04-03 09:00:00Z   2026-04-03 11:00:00Z   Conference room B    confirmed
--------------------------------------------------------------------------------
3 event(s)
```

This is useful when an agent needs to include a calendar snapshot in a text-based report, log, or chat message without parsing JSON. The agent can pipe the output directly into its response or log file.

---

## 18. Protocol Abstraction

**Problem:** Interoperating with external systems often requires understanding complex protocols (email/SMTP, iCal, OAuth handshakes, file format parsing). Agents shouldn't need to implement protocol-level details.

**Pattern:** The API handles all protocol complexity server-side. The agent works with simple JSON fields; the API translates to/from the underlying protocols.

### CalDave example

CalDave abstracts away email and iCal complexity. Agents receive calendar invites via webhook without knowing anything about SMTP, MIME, or iCal parsing.

**Inbound:** Multiple email providers send invites in different formats. CalDave normalizes them in src/routes/inbound.js with a normalizePayload() function that detects the provider format (AgentMail or Postmark), extracts subject, text body, and attachments, and returns a consistent structure. The agent sees a clean webhook payload, never the raw SMTP/MIME format.

The agent sees a clean webhook payload, never the raw SMTP/MIME format:

```json
{
  "type": "event.created",
  "calendar_id": "cal_abc123",
  "event_id": "evt_xyz789",
  "event": {
    "event_id": "evt_xyz789",
    "title": "Project kickoff",
    "start": "2026-04-15T14:00:00.000Z",
    "end": "2026-04-15T15:00:00.000Z",
    "location": "Conference room A",
    "attendees": ["alice@example.com", "bob@example.com"]
  },
  "timestamp": "2026-04-02T10:30:00.000Z"
}
```

**Outbound:** Agents send simple JSON; CalDave handles iCal generation and SMTP:

```bash
# Agent creates event with attendees
curl -s -X POST https://caldave.ai/calendars/cal_abc/events \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Project kickoff",
    "start": "2026-04-15T14:00:00Z",
    "end": "2026-04-15T15:00:00Z",
    "attendees": "alice@example.com, bob@example.com"
  }'
```

CalDave automatically:
1. Generates iCal VCALENDAR format
2. Creates MIME multipart message
3. Sends via SMTP to all attendees
4. Tracks RSVP responses

The agent never sees or handles email/iCal protocols. It works with JSON in, JSON out.

---

## 19. Progressive Enhancement

**Problem:** Forcing agents to configure everything upfront creates a high barrier to entry. Many features might never be needed.

**Pattern:** Start with sensible defaults that work out of the box. Let agents add capabilities incrementally as they need them.

### CalDave example

CalDave follows a progressive enhancement model for calendars:

**Phase 1:** Core functionality works with zero configuration

```bash
curl -s -X POST https://caldave.ai/calendars \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{"name": "Work", "timezone": "America/New_York"}'
# => Works. Events can be created immediately. Invites sent via CalDave's default SMTP.
```

**Phase 2:** Add webhooks when ready for real-time updates

```bash
curl -s -X PATCH https://caldave.ai/calendars/cal_abc \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{"webhook_url": "https://my-agent.com/hooks", "webhook_secret": "s3cret"}'
```

**Phase 3:** Test webhook before relying on it

```bash
curl -s -X POST https://caldave.ai/calendars/cal_abc/webhook/test \
  -H "Authorization: Bearer sk_live_..."
# => { "success": true, "status_code": 200, "message": "Webhook delivered successfully." }
```

**Phase 4:** Configure custom SMTP for branded emails

```bash
curl -s -X PUT https://caldave.ai/agents/smtp \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "host": "smtp.example.com",
    "port": 465,
    "username": "agent@example.com",
    "password": "...",
    "from": "Calendar Agent <agent@example.com>"
  }'
```

**Phase 5:** Test SMTP configuration

```bash
curl -s -X POST https://caldave.ai/agents/smtp/test \
  -H "Authorization: Bearer sk_live_..."
# => { "success": true, "message": "Test email sent successfully to agent@example.com" }
```

**Phase 6:** Remove custom SMTP to revert to defaults

```bash
curl -s -X DELETE https://caldave.ai/agents/smtp \
  -H "Authorization: Bearer sk_live_..."
# => Calendar invites now use CalDave's default SMTP again
```

Key principle: everything works at phase 1. Each subsequent phase adds capability but is never required. An agent can operate indefinitely with just name + timezone on its calendar.

---

## 20. Agent-First Ownership Model

**Problem:** Traditional APIs are human-first: a human creates an account, generates API keys, and grants them to applications. This doesn't work when the agent needs to be autonomous.

**Pattern:** Agents are the primary citizens. They create themselves, manage their own resources, and operate independently. Humans can optionally "claim" agents after the fact for oversight and key management.

### CalDave example

CalDave inverts the traditional ownership model:

**Step 1:** Agent creates itself (no human involved)

```bash
curl -s -X POST https://caldave.ai/agents \
  -H "Content-Type: application/json" \
  -d '{"name": "Meeting Scheduler Bot"}'
# => { "agent_id": "agt_abc123", "api_key": "sk_live_..." }
```

**Step 2:** Agent operates autonomously

```bash
curl -s -X POST https://caldave.ai/calendars \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{"name": "Team Calendar", "timezone": "America/New_York"}'

curl -s -X POST https://caldave.ai/calendars/cal_xyz/events \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{"title": "Sprint planning", "start": "2026-04-10T09:00:00Z", "end": "2026-04-10T11:00:00Z"}'
```

**Step 3 (optional, later):** Human creates account via web UI

```bash
# Human visits https://caldave.ai/signup and creates account
# Receives human API key: hk_live_...
```

**Step 4 (optional):** Human claims the agent for oversight

```bash
curl -s -X POST https://caldave.ai/agents/claim \
  -H "X-Human-Key: hk_live_..." \
  -H "Content-Type: application/json" \
  -d '{"api_key": "sk_live_the_agents_key"}'
# => { "agent_id": "agt_abc123", "claimed": true, "owned_by": "hum_xyz" }
```

Agents can also be auto-associated at creation time:

```bash
curl -s -X POST https://caldave.ai/agents \
  -H "X-Human-Key: hk_live_..." \
  -H "Content-Type: application/json" \
  -d '{"name": "Meeting Scheduler Bot"}'
# => { "agent_id": "agt_abc123", "api_key": "sk_live_...", "owned_by": "hum_xyz" }
```

The implementation in src/routes/agents.js checks for req.human after creating the agent, and if present, inserts a record in the human_agents junction table and includes owned_by in the response.

This inverts the traditional model: the agent is the primary entity, human oversight is optional and additive.

---

## 21. Rate Limiting with Standard Headers

**Problem:** Agents need to know rate limits programmatically to implement backoff. Custom rate limit headers or undocumented limits lead to hard-to-debug 429 errors.

**Pattern:** Use RFC draft-7 standard headers on every response. Apply different limits to different operation types.

### CalDave example

CalDave uses express-rate-limit with RFC draft-7 standard headers configured in src/middleware/rateLimit.js. Different limiters are defined for general API calls (1000/minute), agent creation (20/hour), inbound webhooks (60/minute), and human authentication (10/15 minutes). Each limiter uses standardHeaders: 'draft-7' to include RateLimit-Limit, RateLimit-Remaining, and RateLimit-Reset headers, and disables legacy X-RateLimit-* headers.

Every response includes standard headers:

```bash
curl -i https://caldave.ai/calendars \
  -H "Authorization: Bearer sk_live_..."

HTTP/2 200
ratelimit-limit: 1000
ratelimit-remaining: 997
ratelimit-reset: 45
content-type: application/json
```

Headers:
- `RateLimit-Limit` -- total requests allowed in window (1000)
- `RateLimit-Remaining` -- requests remaining in current window (997)
- `RateLimit-Reset` -- seconds until window resets (45)

An agent can read these on every response and implement intelligent backoff by checking the remaining count and reset time.

CalDave applies different rate limits to different operation types:

| Operation | Limit | Window | Rationale |
|---|---|---|---|
| General API | 1000 requests | per minute per IP | Normal operations |
| Agent creation | 20 requests | per hour per IP | Prevent abuse |
| Inbound webhooks | 60 requests | per minute per IP | Email volume |
| Human auth | 10 requests | per 15 minutes per IP | Prevent brute force |

---

## 22. MCP Support

**Problem:** HTTP APIs require agents to construct URLs, set headers, format JSON bodies, and parse responses. AI agents that support MCP (Model Context Protocol) can use structured tool calls instead, which is more natural for LLMs.

**Pattern:** Expose the same API capabilities through MCP tool calls, in addition to the HTTP API. Offer both remote (HTTP transport) and local (STDIO) modes.

### CalDave example

CalDave provides a remote MCP endpoint at `POST /mcp` implemented in src/routes/mcp.mjs. The MCP server provides 24 tools covering all CalDave operations (agent management, calendars, events, feeds, SMTP), a guide resource at caldave://guide with getting-started content, and structured instructions with workflow descriptions and tool selection guidance.

Tools are registered in src/lib/mcp-tools.mjs with name, description, and JSON Schema for input validation. Each tool calls the underlying HTTP API and returns formatted results.

Agent configuration in Claude Desktop or other MCP clients:

```json
{
  "mcpServers": {
    "caldave": {
      "url": "https://caldave.ai/mcp",
      "headers": {
        "Authorization": "Bearer sk_live_..."
      }
    }
  }
}
```

Once configured, the agent can use natural language tool calls. For example, an agent can say "I need to create a calendar called Team Events in Pacific time" and the MCP client will translate that to a caldave_create_calendar tool call with appropriate parameters. The tool response includes the full calendar object with calendar_id, email address, feed URL, and other details.

MCP is additive. The HTTP API remains the primary interface. MCP provides an alternative integration path for agents that support it, with the same capabilities but a more natural tool-based interface for LLMs.

---

## 23. Seed Data on Resource Creation

**Problem:** After creating a new container resource (a calendar), it's empty. An agent can't test retrieval without first creating child resources (events). This adds an extra step to the bootstrapping flow.

**Pattern:** Automatically create a sample resource inside new containers so the agent can immediately test listing and retrieval. Allow opting out for production use.

### CalDave example

When an agent creates a calendar, CalDave automatically creates a welcome event unless welcome_event is explicitly set to false. The implementation in src/routes/calendars.js creates an event scheduled for 9am tomorrow in the calendar's timezone using Postgres AT TIME ZONE conversion. The event asks the agent to send feedback to the founder with details about their use case.

Default behavior -- welcome event created:

```bash
curl -s -X POST https://caldave.ai/calendars \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{"name": "Work", "timezone": "America/New_York"}'

# Immediately check for events
curl -s https://caldave.ai/calendars/cal_abc123/upcoming \
  -H "Authorization: Bearer sk_live_..."
# => Returns the welcome event at 9am tomorrow
```

Production use -- skip welcome event:

```bash
curl -s -X POST https://caldave.ai/calendars \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{"name": "Work", "timezone": "America/New_York", "welcome_event": false}'

# Calendar starts empty
```

The welcome event serves three purposes:
1. Lets agents test event retrieval immediately after calendar creation
2. Provides a friendly onboarding message asking for product feedback
3. Demonstrates how events work (title, description, time formatting)

Agents building integrations can leave the default behavior. Production deployments can set `welcome_event: false`.

---

## 24. Pre-Computed URLs in Responses

**Problem:** After creating a resource, the agent needs to know derived URLs (feed URLs, webhook URLs, email addresses). Constructing these from component parts is error-prone.

**Pattern:** Resource creation responses include fully assembled URLs that are ready to use.

### CalDave example

Calendar creation returns all derived URLs pre-computed in src/routes/calendars.js. The response includes the calendar email address (constructed from calendar ID), the iCal feed URL (with feed token as query parameter), the feed token separately, and the inbound webhook URL for email forwarding. All URLs are fully qualified and ready to use without any string manipulation.

Response:

```json
{
  "calendar_id": "cal_a1b2c3XyZw",
  "name": "Work",
  "timezone": "America/New_York",
  "email": "cal-a1b2c3XyZw@invite.caldave.ai",
  "ical_feed_url": "https://caldave.ai/feeds/cal_a1b2c3XyZw.ics?token=feed_def456...",
  "feed_token": "feed_def456...",
  "inbound_webhook_url": "https://caldave.ai/inbound/inb_ghi789...",
  "message": "This calendar can receive invites at cal-a1b2c3XyZw@invite.caldave.ai. Forward emails to https://caldave.ai/inbound/inb_ghi789.... Save this information."
}
```

The agent gets:
- **email** -- ready to give to users or other systems for sending invites
- **ical_feed_url** -- ready to subscribe to in any calendar app
- **feed_token** -- if it needs to construct feed URLs manually
- **inbound_webhook_url** -- ready to configure in email forwarding services

The agent doesn't need to know URL structures, token formats, or domain names. It just uses the URLs directly from the response.

This pattern appears throughout CalDave:
- Event responses include `ical_uid` for external system integration
- Feed responses include `ical_feed_url` assembled with authentication token
- Error responses include links to documentation
- `/man` response includes `base_url` and fully qualified endpoint paths

---

## 25. Fire-and-Forget Webhook Delivery

**Problem:** Webhook delivery shouldn't slow down API responses. If the agent's webhook receiver is down or slow, the API response to the original request shouldn't be delayed.

**Pattern:** Wrap webhook delivery in an async IIFE that catches and logs all errors. Call it after the response is sent.

### CalDave example

CalDave's `fireWebhook()` function in src/lib/webhooks.js is wrapped in an async IIFE that runs independently of the response. The entire function is fire-and-forget: it queries for webhook configuration, constructs the payload, adds HMAC signature if configured, sends the request with a 10-second timeout, logs success or failure, and catches all errors to prevent crashes. The catch handler logs errors but never throws, ensuring webhook failures can't impact the application.

Used in route handlers -- response is sent FIRST, webhook fires after. For example, in src/routes/events.js, the POST handler creates the event, formats the response, sends it to the client with res.status(201).json(), and then calls fireWebhook() asynchronously.

Key details:
- Async IIFE `(async () => { ... })()` runs independently
- `.catch()` swallows all errors -- never crashes the app
- 10-second timeout prevents hanging connections
- Errors logged to console but never returned to client
- Called AFTER `res.json()` -- webhook delivery can't delay the API response

This pattern ensures webhook delivery failures (network issues, slow receivers, 500 errors) never impact API performance or reliability. The agent gets a fast response regardless of webhook state.

---

## Summary: Design Principles

These 25 patterns are organized around 7 core principles:

1. **Self-bootstrapping.** An agent can go from zero knowledge to full API usage by following the chain: hit any URL, get pointed to `/man`, follow progressive recommendations. No human setup required.

2. **Mistake tolerance.** Unknown fields are rejected with specific messages. Type-prefixed IDs prevent cross-entity confusion. The 404 handler points to documentation. Aliases are accepted for commonly confused field names.

3. **Token efficiency.** Topic filtering and guide mode reduce payload size. Pre-computed URLs eliminate the need to construct URLs. ISO 8601 durations are machine-parseable in a single line of code.

4. **Proactive notifications.** Webhooks deliver changes to agents without polling. Time-based webhooks handle scheduled triggers. The changelog notifies agents of new features.

5. **Protocol abstraction.** External protocol complexity (email, iCal, file formats, SMTP) is handled server-side. Agents work with JSON; the server handles the rest.

6. **Progressive enhancement.** Agents start with zero configuration and add capabilities (webhooks, custom SMTP, human ownership) as needed. No feature requires upfront setup.

7. **Machine-parseable everything.** JSON responses, ISO 8601 durations, standard rate limit headers, consistent error format, type-prefixed IDs. Every response is designed to be parsed by code, not read by humans.

---

## About CalDave

CalDave is a production calendar-as-a-service API designed from the ground up for AI agent consumers. It implements all 25 patterns described in this document.

- **Website:** https://caldave.ai
- **Documentation:** https://caldave.ai/docs
- **API Manual:** https://caldave.ai/man
- **Source:** The patterns in this document are extracted from CalDave's real production codebase

CalDave demonstrates that agent-first API design isn't theoretical -- it works in production, scales to real workloads, and provides a better developer experience for both AI agents and human developers.
