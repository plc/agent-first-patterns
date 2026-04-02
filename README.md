# Agent-First API Design Patterns

A detailed reference of every design pattern used in CalDave that makes it an "agent-first" web application. Each pattern includes the rationale, implementation details, and concrete API call examples. These patterns are transferable to any API built for AI agent consumers.

---

## Table of Contents

1. [Self-Describing API Manual (`GET /man`)](#1-self-describing-api-manual)
2. [Progressive Recommendation Engine](#2-progressive-recommendation-engine)
3. [Soft Authentication](#3-soft-authentication)
4. [Zero-Auth Agent Bootstrapping](#4-zero-auth-agent-bootstrapping)
5. [Context Window Management (Topic Filtering and Guide Mode)](#5-context-window-management)
6. [Generated Curl Examples with Real Credentials](#6-generated-curl-examples-with-real-credentials)
7. [Agent-Aware Changelog](#7-agent-aware-changelog)
8. [Unknown Field Rejection](#8-unknown-field-rejection)
9. [Type-Prefixed IDs](#9-type-prefixed-ids)
10. [Consistent Error Format](#10-consistent-error-format)
11. [404 as Discovery Mechanism](#11-404-as-discovery-mechanism)
12. [Agent-Scoped Error Introspection](#12-agent-scoped-error-introspection)
13. [Polling Helpers (ISO 8601 Durations)](#13-polling-helpers)
14. [Webhook System (Mutation + Time-Based)](#14-webhook-system)
15. [Field Alias Acceptance](#15-field-alias-acceptance)
16. [Arbitrary Metadata Field](#16-arbitrary-metadata-field)
17. [Plain Text Rendering](#17-plain-text-rendering)
18. [Protocol Abstraction (Email, iCal, RSVP)](#18-protocol-abstraction)
19. [Progressive Enhancement (SMTP, Webhooks)](#19-progressive-enhancement)
20. [Agent-First Ownership Model](#20-agent-first-ownership-model)
21. [Rate Limiting with Standard Headers](#21-rate-limiting-with-standard-headers)
22. [MCP (Model Context Protocol) Support](#22-mcp-support)
23. [Welcome Event (Non-Empty Initial State)](#23-welcome-event)
24. [Pre-Computed URLs in Responses](#24-pre-computed-urls-in-responses)
25. [iCal Feed with Caching Headers](#25-ical-feed-with-caching-headers)
26. [Inbound Email Normalization](#26-inbound-email-normalization)
27. [Fire-and-Forget Webhook Delivery](#27-fire-and-forget-webhook-delivery)
28. [Recurring Event Abstraction](#28-recurring-event-abstraction)

---

## 1. Self-Describing API Manual

**File:** `src/routes/man.js`

**Problem:** AI agents have no prior knowledge of your API. Traditional docs are HTML pages designed for human eyes. An agent needs to discover your API's capabilities, authenticate, and start making calls -- all programmatically.

**Pattern:** Expose a single endpoint that returns the complete API reference as structured JSON. An agent can go from zero knowledge to full API usage with one HTTP call.

**Implementation:**

```
GET /man
```

No authentication required. Returns:

```json
{
  "overview": "CalDave is a calendar-as-a-service API for AI agents...",
  "base_url": "https://caldave.ai",
  "available_topics": ["agents", "smtp", "calendars", "events", "feeds", "errors"],
  "rate_limits": {
    "api": "1000 requests/minute per IP",
    "agent_creation": "20 requests/hour per IP",
    "headers": "RateLimit-Limit, RateLimit-Remaining, RateLimit-Reset (RFC draft-7)"
  },
  "error_format": {
    "shape": "{ \"error\": \"Human-readable message\" }",
    "status_codes": {
      "200": "Success",
      "400": "Validation error -- check the error message for details",
      "401": "Missing or invalid API key",
      "404": "Resource not found",
      "429": "Rate limited -- check RateLimit-Reset header",
      "500": "Server error -- retry or check GET /errors"
    }
  },
  "webhook_verification": {
    "header": "X-CalDave-Signature",
    "algorithm": "HMAC-SHA256(webhook_secret, raw_request_body)",
    "format": "hex digest"
  },
  "webhook_event_types": {
    "mutation_based": ["event.created", "event.updated", "event.deleted", "event.responded"],
    "time_based": ["event.starting", "event.ending"]
  },
  "your_context": {
    "authenticated": false,
    "agent_id": null,
    "calendars": []
  },
  "recommended_next_step": { ... },
  "endpoints": [
    {
      "method": "POST",
      "path": "/agents",
      "description": "Create a new agent identity...",
      "auth": "none",
      "parameters": [ ... ],
      "example_curl": "curl -s -X POST \"https://caldave.ai/agents\" ...",
      "example_response": { ... }
    }
  ]
}
```

Every endpoint entry includes: method, path, description, auth requirement, parameter list (with name, location, type, required flag, description), a generated curl command, and an example response.

**Key detail:** The endpoint catalog is pre-computed at module load time since it is static:

```javascript
const CACHED_ENDPOINTS = getEndpoints();
```

**API call to test:**

```bash
# Unauthenticated -- discover the full API
curl -s https://caldave.ai/man | jq '.endpoints | length'

# Authenticated -- get personalized context
curl -s https://caldave.ai/man \
  -H "Authorization: Bearer sk_live_abc123..."
```

---

## 2. Progressive Recommendation Engine

**File:** `src/routes/man.js` (function `buildRecommendation`)

**Problem:** An agent that reads 30 endpoints doesn't know which one to call first. Without guidance, agents make random or incorrect choices about what to do next.

**Pattern:** Inspect the agent's current state (authenticated? named? has calendars? has events? claimed by a human?) and return the single most important next action with a ready-to-execute curl command.

**Implementation:** The recommendation chain:

| Agent State | Recommendation | Endpoint |
|---|---|---|
| Not authenticated | "Create an agent" | `POST /agents` |
| No name set | "Name your agent" | `PATCH /agents` |
| No calendars | "Create a calendar" | `POST /calendars` |
| No events (only welcome events) | "Create an event" | `POST /calendars/:id/events` |
| Not claimed by a human | "Claim this agent" | `POST /agents/claim` |
| Everything done | "Check upcoming events" | `GET /calendars/:id/upcoming` |

The recommendation always includes an `action` (what to do), `description` (why), `endpoint` (which endpoint), and `curl` (a ready-to-paste command with the agent's real IDs interpolated).

**Code pattern:**

```javascript
function buildRecommendation(context, apiKey, calId) {
  if (!context.authenticated) {
    return {
      action: 'Create an agent',
      description: 'You are not authenticated. Create an agent to get an API key...',
      endpoint: 'POST /agents',
      curl: buildCurl('POST', '/agents', {
        body: { name: 'My Agent', description: 'Brief description...' },
      }),
    };
  }

  if (!context.agent_name) {
    return {
      action: 'Name your agent',
      endpoint: 'PATCH /agents',
      curl: buildCurl('PATCH', '/agents', { apiKey, body: { name: 'My Agent' } }),
    };
  }

  if (context.calendars.length === 0) { ... }
  // ...and so on down the chain
}
```

**API call showing progression:**

```bash
# Step 1: Agent sees "Create an agent"
curl -s https://caldave.ai/man?guide | jq '.recommended_next_step'
# => { "action": "Create an agent", "curl": "curl -s -X POST ..." }

# Step 2: After creating agent, agent sees "Name your agent"
curl -s https://caldave.ai/man?guide \
  -H "Authorization: Bearer sk_live_abc123..." | jq '.recommended_next_step'
# => { "action": "Name your agent", "curl": "curl -s -X PATCH ..." }

# Step 3: After naming, agent sees "Create a calendar"
# ...and so on
```

---

## 3. Soft Authentication

**Files:** `src/routes/man.js`, `src/routes/changelog.js`

**Problem:** Discovery endpoints need to work without authentication (so new agents can find out how to create credentials), but authenticated agents should get personalized data. Standard auth middleware returns 401 for missing tokens, blocking unauthenticated access entirely.

**Pattern:** "Soft auth" resolves a Bearer token when present but never returns 401. If the token is valid, `req.agent` is populated. If missing or invalid, `req.agent` is set to `null` and the request continues.

**Implementation:**

```javascript
async function softAuth(req, res, next) {
  const header = req.headers.authorization;
  if (!header || !header.startsWith('Bearer ')) {
    req.agent = null;
    return next();   // No 401 -- just proceed without agent context
  }
  const token = header.slice(7);
  const hash = hashKey(token);
  try {
    const result = await pool.query(
      'SELECT id, name, description FROM agents WHERE api_key_hash = $1',
      [hash]
    );
    req.agent = result.rows.length > 0 ? result.rows[0] : null;
  } catch {
    req.agent = null;  // DB errors don't block the request
  }
  next();
}
```

**Where to use soft auth vs hard auth:**

| Endpoint type | Auth style | Rationale |
|---|---|---|
| Discovery (`/man`, `/changelog`) | Soft auth | Must be accessible to unauthenticated agents |
| Resource CRUD (`/calendars`, `/events`) | Hard auth (401) | Data must be scoped to the authenticated agent |
| Agent creation (`POST /agents`) | No auth | This is how agents get credentials in the first place |
| Feeds (`/feeds/:id.ics`) | Token in URL | No headers needed for calendar client subscriptions |
| Inbound email (`/inbound/:token`) | Token in URL | Webhook providers can't set Bearer headers |

---

## 4. Zero-Auth Agent Bootstrapping

**File:** `src/routes/agents.js`

**Problem:** Traditional APIs require a human to create an account, log in, navigate to a dashboard, generate an API key, and configure it. AI agents can't do any of that.

**Pattern:** A single unauthenticated POST creates an agent identity and returns a usable API key. The agent goes from zero credentials to fully authenticated in one HTTP call.

**Implementation:**

```bash
curl -s -X POST https://caldave.ai/agents \
  -H "Content-Type: application/json" \
  -d '{"name": "My Agent", "description": "Manages team meetings"}'
```

Response:

```json
{
  "agent_id": "agt_x7y8z9AbCd",
  "api_key": "sk_live_...",
  "name": "My Agent",
  "description": "Manages team meetings",
  "message": "Store these credentials securely. The API key will not be shown again."
}
```

Key details:
- The API key is shown exactly once. Only the SHA-256 hash is stored.
- The warning message is itself agent-friendly: it tells the agent exactly what to do with the response.
- `name` and `description` are optional, so the absolute minimum bootstrapping call is just `POST /agents` with an empty body.
- The endpoint is rate-limited (20/hour per IP) to prevent abuse.

**Complete bootstrapping sequence in 3 calls:**

```bash
# 1. Create agent
RESPONSE=$(curl -s -X POST https://caldave.ai/agents \
  -H "Content-Type: application/json" \
  -d '{"name": "My Agent"}')
API_KEY=$(echo $RESPONSE | jq -r '.api_key')

# 2. Create calendar
curl -s -X POST https://caldave.ai/calendars \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Work", "timezone": "America/New_York"}'

# 3. Create first event
curl -s -X POST https://caldave.ai/calendars/cal_abc123/events \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title": "Standup", "start": "2026-04-03T09:00:00-04:00", "end": "2026-04-03T09:15:00-04:00"}'
```

---

## 5. Context Window Management

**File:** `src/routes/man.js`

**Problem:** AI agents have limited context windows. The full `/man` response with 30+ endpoints can be thousands of tokens. An agent working on events doesn't need to see SMTP configuration endpoints.

**Pattern:** Two mechanisms to reduce payload size: topic filtering and guide mode.

**Topic filtering** (`?topic=`): Returns only endpoints matching the specified categories. Discovery endpoints (`/man`, `/changelog`) are always included regardless of filter.

```bash
# Only see event-related endpoints
curl -s "https://caldave.ai/man?topic=events" \
  -H "Authorization: Bearer $API_KEY" | jq '.endpoints | length'
# => 8 (vs 30+ for the full catalog)

# Multiple topics
curl -s "https://caldave.ai/man?topic=events,calendars" \
  -H "Authorization: Bearer $API_KEY"

# Invalid topic returns a helpful error
curl -s "https://caldave.ai/man?topic=bogus"
# => { "error": "Unknown topic: bogus. Available: agents, smtp, calendars, events, feeds, errors" }
```

Available topics: `agents`, `smtp`, `calendars`, `events`, `feeds`, `errors`

**Guide mode** (`?guide`): Returns only the overview, agent context, and recommended next step. No endpoint catalog at all.

```bash
curl -s "https://caldave.ai/man?guide" \
  -H "Authorization: Bearer $API_KEY"
```

Response (much smaller):

```json
{
  "overview": "CalDave is a calendar-as-a-service API for AI agents...",
  "base_url": "https://caldave.ai",
  "rate_limits": { "api": "1000/min", "agent_creation": "20/hour" },
  "error_format": { ... },
  "your_context": {
    "authenticated": true,
    "agent_id": "agt_abc123",
    "calendars": [{ "id": "cal_xyz", "name": "Work", "event_count": 12 }]
  },
  "recommended_next_step": { ... },
  "discover_more": {
    "full_api_reference": "GET https://caldave.ai/man (without ?guide)...",
    "changelog": "GET https://caldave.ai/changelog..."
  }
}
```

---

## 6. Generated Curl Examples with Real Credentials

**File:** `src/routes/man.js` (function `buildCurl`)

**Problem:** Example code in docs typically uses placeholder values like `YOUR_API_KEY` and `YOUR_CALENDAR_ID`. An agent has to figure out what to substitute. This is an error-prone step that wastes tokens.

**Pattern:** When the agent is authenticated, every curl example in the `/man` response is generated with the agent's actual API key and real calendar IDs.

**Implementation:**

```javascript
function buildCurl(method, path, { apiKey, calId, evtId, body } = {}) {
  let resolved = path
    .replace(':id', calId || 'CAL_ID')
    .replace(':event_id', evtId || 'EVT_ID');

  let url = `${BASE}${resolved}`;
  const parts = [];
  if (method === 'GET') {
    parts.push(`curl -s "${url}"`);
  } else {
    parts.push(`curl -s -X ${method} "${url}"`);
  }
  if (apiKey) parts.push(`-H "Authorization: Bearer ${apiKey}"`);
  if (body) {
    parts.push('-H "Content-Type: application/json"');
    parts.push(`-d '${JSON.stringify(body)}'`);
  }
  return parts.join(' \\\n  ');
}
```

When unauthenticated, examples use `YOUR_API_KEY` and `CAL_ID`. When authenticated, they use the real values. This means the agent can copy any curl command from the `/man` response and execute it directly.

---

## 7. Agent-Aware Changelog

**File:** `src/routes/changelog.js`

**Problem:** Agents don't know when new features are added to the API. They continue using the same endpoints they were originally programmed with, missing improvements.

**Pattern:** A structured changelog endpoint that, when authenticated, splits entries into "changes since your agent was created" vs "changes before you existed." Includes personalized recommendations and explicit polling guidance.

**API call:**

```bash
curl -s https://caldave.ai/changelog \
  -H "Authorization: Bearer $API_KEY"
```

**Authenticated response:**

```json
{
  "description": "CalDave API changelog...",
  "poll_recommendation": "Check this endpoint approximately once per week.",
  "docs_url": "https://caldave.ai/docs",
  "total_changes": 28,
  "your_agent": {
    "agent_id": "agt_abc123",
    "name": "My Agent",
    "created_at": "2026-02-20T10:00:00.000Z"
  },
  "changes_since_signup": [
    {
      "date": "2026-03-31",
      "changes": [
        {
          "type": "feature",
          "title": "Time-based webhooks: event.starting and event.ending",
          "description": "New webhook types...",
          "endpoints": [],
          "docs": "https://caldave.ai/docs#webhook-events"
        }
      ]
    }
  ],
  "changes_since_signup_count": 3,
  "changelog": [ /* older entries */ ],
  "recommendations": [
    {
      "action": "Create an event",
      "why": "You have 1 calendar but haven't created any events yet.",
      "how": "POST /calendars/cal_abc123/events with {\"title\": \"...\", \"start\": \"...\", \"end\": \"...\"}",
      "docs": "https://caldave.ai/docs#events"
    }
  ]
}
```

**Unauthenticated response** (simpler):

```json
{
  "description": "CalDave API changelog...",
  "poll_recommendation": "Check this endpoint approximately once per week.",
  "tip": "Pass your API key as a Bearer token to see which changes are new since your agent was created.",
  "changelog": [ /* all entries */ ]
}
```

The `recommendations` array is computed by inspecting the agent's state (same pattern as `/man` but with more fields: `action`, `why`, `how`, `docs`).

Each changelog entry has a `type` field (`feature`, `improvement`, `fix`) so agents can programmatically filter for what matters to them.

---

## 8. Unknown Field Rejection

**Files:** `src/routes/events.js`, `src/routes/agents.js`, `src/routes/calendars.js`

**Problem:** LLMs frequently hallucinate field names. If `start_time` is the internal column name but the API accepts `start`, an agent might send `start_time` and the API would silently ignore it. The agent would think it set the start time, but nothing happened.

**Pattern:** Every write endpoint validates incoming fields against a known set and returns a specific error listing the unknown fields. No field is ever silently ignored.

**Implementation:**

```javascript
const KNOWN_EVENT_FIELDS = new Set([
  'title', 'start', 'end', 'description', 'metadata', 'location',
  'status', 'attendees', 'recurrence', 'rrule', 'all_day',
]);

function checkUnknownFields(body, knownFields) {
  const unknown = Object.keys(body).filter(k => !knownFields.has(k));
  if (unknown.length === 0) return null;
  return `Unknown field${unknown.length > 1 ? 's' : ''}: ${unknown.join(', ')}`;
}

// In route handler:
const unknownErr = checkUnknownFields(req.body, KNOWN_EVENT_FIELDS);
if (unknownErr) return res.status(400).json({ error: unknownErr });
```

**Example of the error an agent receives:**

```bash
curl -s -X POST https://caldave.ai/calendars/cal_abc/events \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title": "Test", "start_time": "2026-04-01T10:00:00Z", "end_time": "2026-04-01T11:00:00Z"}'
```

Response (400):

```json
{
  "error": "Unknown fields: start_time, end_time"
}
```

The agent immediately knows the correct field names are `start` and `end`, not `start_time` and `end_time`.

This pattern is applied everywhere:
- `POST /agents` -- allowed: `name`, `description`
- `PATCH /agents` -- allowed: `name`, `description`
- `POST /calendars` -- allowed: `name`, `timezone`, `webhook_url`, `webhook_secret`, `agentmail_api_key`, `welcome_event`
- `POST /calendars/:id/events` -- allowed: `title`, `start`, `end`, `description`, `metadata`, `location`, `status`, `attendees`, `recurrence`, `rrule`, `all_day`
- `PUT /agents/smtp` -- allowed: `host`, `port`, `username`, `password`, `from`, `secure`

---

## 9. Type-Prefixed IDs

**File:** `src/lib/ids.js`

**Problem:** APIs that use UUIDs or numeric IDs make it easy to pass a calendar ID where an event ID is expected. The error message ("not found") doesn't tell the agent it used the wrong type of ID.

**Pattern:** Every ID in the system includes a prefix that identifies its entity type. An agent (or a human reviewing logs) can immediately tell what kind of entity an ID refers to.

**Implementation:**

```javascript
const { customAlphabet } = require('nanoid');

const alphabet = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz';
const shortId = customAlphabet(alphabet, 12);
const longId = customAlphabet(alphabet, 32);

const agentId     = () => `agt_${shortId()}`;       // agt_x7y8z9AbCd
const calendarId  = () => `cal_${shortId()}`;       // cal_a1b2c3XyZ
const eventId     = () => `evt_${shortId()}`;       // evt_p4q5r6StUv
const feedToken   = () => `feed_${longId()}`;       // feed_abc123...def456 (32 chars)
const inboundToken = () => `inb_${longId()}`;       // inb_abc123...def456
const apiKey      = () => `sk_live_${longId()}`;    // sk_live_abc123...def456
const humanId     = () => `hum_${shortId()}`;       // hum_g7h8i9JkLm
const humanApiKey = () => `hk_live_${longId()}`;    // hk_live_abc123...def456
const sessionToken = () => `sess_${longId()}`;      // sess_abc123...def456
```

| Prefix | Entity | Length |
|---|---|---|
| `agt_` | Agent | 16 chars |
| `cal_` | Calendar | 16 chars |
| `evt_` | Event | 16 chars |
| `feed_` | iCal feed token | 36 chars |
| `inb_` | Inbound email token | 36 chars |
| `sk_live_` | Agent API key | 40 chars |
| `hk_live_` | Human API key | 40 chars |
| `hum_` | Human account | 16 chars |
| `sess_` | Session | 36 chars |

Design decisions:
- Short IDs (12 random chars) for entities that appear in URLs and logs
- Long IDs (32 random chars) for secrets and tokens
- `sk_live_` prefix convention matches Stripe's well-known pattern, which LLMs are trained on
- Alphanumeric alphabet (no special chars) so IDs are URL-safe without encoding

---

## 10. Consistent Error Format

**Files:** All route files

**Problem:** Inconsistent error formats force agents to handle multiple shapes. Some APIs return `{ "message": "..." }`, others `{ "errors": [...] }`, others `{ "code": 123, "detail": "..." }`.

**Pattern:** Every error response in the entire API uses the same shape: `{ "error": "Human-readable message" }`. This is documented in the `/man` response so agents know what to expect before they encounter errors.

**Examples across the API:**

```json
// 400 - Validation error
{ "error": "title is required" }

// 400 - Unknown fields
{ "error": "Unknown fields: start_time, end_time" }

// 400 - Type error
{ "error": "name must be a string" }

// 400 - Size limit
{ "error": "description exceeds 64KB limit" }

// 401 - Auth error
{ "error": "Missing or invalid Authorization header" }

// 404 - Not found
{ "error": "Calendar not found" }

// 404 - Global catch-all
{ "error": "Not found. Try GET /man for the API reference." }

// 429 - Rate limit
{ "error": "Too many requests, please try again later" }

// 500 - Server error
{ "error": "Failed to create event" }
```

The error message is always a single string, not a code, not an array, not a nested object. Agent code to handle any error is always:

```javascript
const data = await response.json();
if (data.error) {
  console.log("API error:", data.error);
}
```

---

## 11. 404 as Discovery Mechanism

**File:** `src/index.js`

**Problem:** When an agent hits a wrong URL, it gets a generic "Not Found" and has no idea where to go next.

**Pattern:** The 404 catch-all response points the agent to the API manual:

```javascript
app.use((req, res) => {
  res.status(404).json({ error: 'Not found. Try GET /man for the API reference.' });
});
```

Every wrong URL becomes a learning opportunity. The agent can immediately call `GET /man` to discover the correct endpoints.

---

## 12. Agent-Scoped Error Introspection

**Files:** `src/lib/errors.js`, `src/index.js`

**Problem:** When an agent triggers a server error (500), traditional APIs just say "Internal server error." The agent can't debug the issue, and a human has to check server logs.

**Pattern:** All server errors are logged to an `error_log` database table with the agent's ID attached. Agents can query their own errors via `GET /errors`.

**Error logging utility:**

```javascript
async function logError(err, ctx = {}) {
  const message = err instanceof Error ? err.message : String(err);
  const stack = err instanceof Error ? err.stack : null;

  await pool.query(
    `INSERT INTO error_log (route, method, status_code, message, stack, agent_id)
     VALUES ($1, $2, $3, $4, $5, $6)`,
    [ctx.route, ctx.method, ctx.status_code || 500, message, stack, ctx.agent_id]
  );
}
```

Every catch block in the application calls `logError` with context:

```javascript
catch (err) {
  logError(err, { route: 'POST /calendars/:id/events', method: 'POST', agent_id: req.agent?.id });
  res.status(500).json({ error: 'Failed to create event' });
}
```

**API calls to debug:**

```bash
# List recent errors
curl -s https://caldave.ai/errors \
  -H "Authorization: Bearer $API_KEY"

# Response:
{
  "errors": [
    {
      "id": 42,
      "route": "POST /calendars/:id/events",
      "method": "POST",
      "status_code": 500,
      "message": "invalid input syntax for type timestamp...",
      "agent_id": "agt_abc123",
      "created_at": "2026-04-01T15:30:00.000Z"
    }
  ],
  "count": 1
}

# Get full stack trace for a specific error
curl -s https://caldave.ai/errors/42 \
  -H "Authorization: Bearer $API_KEY"

# Filter by route
curl -s "https://caldave.ai/errors?route=events" \
  -H "Authorization: Bearer $API_KEY"
```

Errors are scoped: an agent can only see errors associated with its own `agent_id`. This lets agents self-diagnose without exposing other agents' errors.

---

## 13. Polling Helpers

**File:** `src/routes/events.js`

**Problem:** An agent that polls for upcoming events on a fixed interval (say, every 5 minutes) wastes API calls when the next event is 6 hours away, and misses events when the next one is 30 seconds away.

**Pattern:** The `GET /calendars/:id/upcoming` endpoint returns a `next_event_starts_in` field as an ISO 8601 duration string, telling the agent exactly how long until the next event.

**Implementation:**

```javascript
function msToIsoDuration(ms) {
  if (ms <= 0) return 'PT0S';
  const totalSeconds = Math.floor(ms / 1000);
  const hours = Math.floor(totalSeconds / 3600);
  const minutes = Math.floor((totalSeconds % 3600) / 60);
  const seconds = totalSeconds % 60;

  let parts = 'PT';
  if (hours > 0) parts += `${hours}H`;
  if (minutes > 0) parts += `${minutes}M`;
  if (seconds > 0 || parts === 'PT') parts += `${seconds}S`;
  return parts;
}
```

**API call:**

```bash
curl -s https://caldave.ai/calendars/cal_abc123/upcoming \
  -H "Authorization: Bearer $API_KEY"
```

**Response:**

```json
{
  "events": [
    {
      "id": "evt_xyz789",
      "title": "Team standup",
      "start": "2026-04-02T14:00:00.000Z",
      "end": "2026-04-02T14:15:00.000Z",
      "status": "confirmed"
    }
  ],
  "next_event_starts_in": "PT14M30S",
  "timezone": "America/New_York"
}
```

The agent can now schedule its next poll for ~14 minutes from now instead of using a fixed interval. `PT14M30S` is an ISO 8601 duration, parseable by standard libraries in any language.

The changelog endpoint also includes explicit polling guidance:

```json
{
  "poll_recommendation": "Check this endpoint approximately once per week."
}
```

---

## 14. Webhook System

**Files:** `src/lib/webhooks.js`, `src/lib/webhook-poller.js`

**Problem:** Polling wastes resources and introduces latency. Agents need to know immediately when events change, and they need to know when events are about to start or have ended.

**Pattern:** Two categories of webhooks:

**Mutation webhooks** (fired immediately from route handlers):
- `event.created` -- new event added
- `event.updated` -- event details changed
- `event.deleted` -- event cancelled or removed
- `event.responded` -- RSVP response recorded

**Time-based webhooks** (fired by a background poller):
- `event.starting` -- event's start time has arrived
- `event.ending` -- event's end time has arrived

**Setting up webhooks:**

```bash
# Configure webhook URL when creating a calendar
curl -s -X POST https://caldave.ai/calendars \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Work",
    "timezone": "America/New_York",
    "webhook_url": "https://my-agent.example.com/webhooks",
    "webhook_secret": "my-hmac-secret-123"
  }'

# Or add webhooks to an existing calendar
curl -s -X PATCH https://caldave.ai/calendars/cal_abc123 \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "webhook_url": "https://my-agent.example.com/webhooks",
    "webhook_secret": "my-hmac-secret-123"
  }'
```

**Testing the webhook before relying on it:**

```bash
curl -s -X POST https://caldave.ai/calendars/cal_abc123/webhook/test \
  -H "Authorization: Bearer $API_KEY"
```

Response:

```json
{
  "success": true,
  "status_code": 200,
  "webhook_url": "https://my-agent.example.com/webhooks",
  "message": "Webhook delivered successfully."
}
```

**Webhook payload format:**

```json
{
  "type": "event.created",
  "calendar_id": "cal_abc123",
  "event_id": "evt_xyz789",
  "event": {
    "id": "evt_xyz789",
    "title": "Team standup",
    "start": "2026-04-03T09:00:00.000Z",
    "end": "2026-04-03T09:15:00.000Z",
    "status": "confirmed"
  },
  "timestamp": "2026-04-02T18:30:00.000Z"
}
```

**HMAC-SHA256 signature verification** (when `webhook_secret` is set):

```javascript
const crypto = require('crypto');

function verifyWebhook(body, signature, secret) {
  const expected = crypto
    .createHmac('sha256', secret)
    .update(body)  // raw body string, not re-serialized
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected)
  );
}

// In webhook handler:
const signature = req.headers['x-caldave-signature'];
const isValid = verifyWebhook(rawBody, signature, 'my-hmac-secret-123');
```

The webhook delivery implementation is fire-and-forget with a 10-second timeout. It never blocks the API response.

---

## 15. Field Alias Acceptance

**File:** `src/routes/events.js`

**Problem:** LLMs are trained on large corpora where the term "rrule" is far more common than "recurrence" when referring to RFC 5545 recurrence rules. Agents frequently send `rrule` instead of `recurrence`.

**Pattern:** Accept common aliases for fields that agents are likely to get wrong. The normalization happens silently before validation.

**Implementation:**

```javascript
function normalizeBody(body) {
  if (body.rrule !== undefined && body.recurrence === undefined) {
    body.recurrence = body.rrule;
  }
  delete body.rrule;
}
```

Both of these calls produce the same result:

```bash
# Using the canonical field name
curl -s -X POST https://caldave.ai/calendars/cal_abc/events \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title": "Standup", "start": "2026-04-03T09:00:00Z", "end": "2026-04-03T09:15:00Z", "recurrence": "FREQ=DAILY;BYDAY=MO,TU,WE,TH,FR"}'

# Using the alias
curl -s -X POST https://caldave.ai/calendars/cal_abc/events \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title": "Standup", "start": "2026-04-03T09:00:00Z", "end": "2026-04-03T09:15:00Z", "rrule": "FREQ=DAILY;BYDAY=MO,TU,WE,TH,FR"}'
```

Note: `rrule` is included in `KNOWN_EVENT_FIELDS` so it doesn't trigger the unknown field rejection. If `recurrence` takes priority when both are present.

---

## 16. Arbitrary Metadata Field

**File:** `src/routes/events.js`, `src/db.js`

**Problem:** Different agents need to store different auxiliary data on events. One agent might track a Zoom link, another might store a CRM deal ID, another might track preparation notes. A rigid schema can't anticipate all these needs.

**Pattern:** Events have a `metadata` JSONB field that accepts arbitrary structured data up to 16KB. The API stores it as-is and returns it as-is.

```bash
curl -s -X POST https://caldave.ai/calendars/cal_abc/events \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Client call",
    "start": "2026-04-03T14:00:00Z",
    "end": "2026-04-03T14:30:00Z",
    "metadata": {
      "zoom_link": "https://zoom.us/j/123456789",
      "crm_deal_id": "deal_789",
      "prep_notes": "Review Q1 numbers before call",
      "priority": "high"
    }
  }'
```

The metadata is returned in all event responses:

```json
{
  "id": "evt_abc123",
  "title": "Client call",
  "start": "2026-04-03T14:00:00.000Z",
  "end": "2026-04-03T14:30:00.000Z",
  "metadata": {
    "zoom_link": "https://zoom.us/j/123456789",
    "crm_deal_id": "deal_789",
    "prep_notes": "Review Q1 numbers before call",
    "priority": "high"
  }
}
```

---

## 17. Plain Text Rendering

**File:** `src/routes/view.js`

**Problem:** Some agents operate in text-only environments (terminal, chat interfaces, simple HTTP clients). Parsing JSON to produce a human-readable calendar view adds unnecessary complexity.

**Pattern:** `GET /calendars/:id/view` returns a pre-formatted `text/plain` table of upcoming events that works directly with `curl` output.

```bash
curl -s https://caldave.ai/calendars/cal_abc123/view \
  -H "Authorization: Bearer $API_KEY"
```

Response (text/plain):

```
Calendar: Work (America/New_York)

Date        Time          Title              Location
----------- ------------- ------------------ ----------------
2026-04-03  9:00 - 9:15   Team standup       Room 42
2026-04-03  14:00 - 14:30 Client call        Zoom
2026-04-04  10:00 - 11:00 Sprint planning    Conference Room B
```

---

## 18. Protocol Abstraction

**Files:** `src/routes/inbound.js`, `src/lib/outbound.js`, `src/routes/events.js`, `src/routes/feeds.js`

**Problem:** Calendar interop requires understanding multiple complex protocols: iCal (RFC 5545), email (SMTP/MIME), RSVP (METHOD:REPLY), timezone handling, recurrence rule expansion. No agent should need to know any of this.

**Pattern:** The API handles all protocol complexity server-side. The agent works with simple JSON fields; the API translates to/from the underlying protocols.

**Receiving invites (inbound email to JSON):**

A human sends an email invite to `cal-abc123@invite.caldave.ai`. CalDave:
1. Receives the webhook from the email provider (Postmark or AgentMail)
2. Normalizes the payload across providers
3. Finds and parses the .ics attachment
4. Extracts event data (title, times, attendees, organizer, recurrence)
5. Creates an event with `source: 'inbound_email'` and `status: 'tentative'`
6. Fires an `event.created` webhook to the agent

The agent sees:

```json
{
  "type": "event.created",
  "event": {
    "id": "evt_xyz789",
    "title": "Quarterly Review",
    "start": "2026-04-15T10:00:00.000Z",
    "end": "2026-04-15T11:00:00.000Z",
    "status": "tentative",
    "source": "inbound_email",
    "organiser_email": "boss@company.com",
    "attendees": ["agent@invite.caldave.ai", "colleague@company.com"]
  }
}
```

**Responding to invites (JSON to iCal REPLY email):**

```bash
curl -s -X POST https://caldave.ai/calendars/cal_abc/events/evt_xyz789/respond \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"response": "accepted"}'
```

Response:

```json
{
  "id": "evt_xyz789",
  "title": "Quarterly Review",
  "status": "confirmed",
  "response": "accepted",
  "message": "Event accepted",
  "email_sent": true
}
```

Behind the scenes, CalDave constructed a METHOD:REPLY iCal email, set the agent's name in the From header, and sent it to the organizer via SMTP. The agent just sent `{"response": "accepted"}`.

**Sending invites (JSON to iCal REQUEST email):**

```bash
curl -s -X POST https://caldave.ai/calendars/cal_abc/events \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Design review",
    "start": "2026-04-10T15:00:00Z",
    "end": "2026-04-10T16:00:00Z",
    "attendees": ["designer@company.com", "pm@company.com"]
  }'
```

Response:

```json
{
  "id": "evt_new123",
  "title": "Design review",
  "attendees": ["designer@company.com", "pm@company.com"],
  "email_sent": true
}
```

The `email_sent` field confirms the invite was dispatched. CalDave generated a METHOD:REQUEST iCal attachment, set the From header to the calendar's email address (with the agent's name as display name), and sent it to both attendees.

---

## 19. Progressive Enhancement

**Files:** `src/routes/agents.js`, `src/routes/calendars.js`

**Problem:** Forcing agents to configure everything upfront (SMTP, webhooks, email forwarding) creates a high barrier to entry. Many features might never be needed.

**Pattern:** Start with sensible defaults, let agents add capabilities incrementally.

**SMTP example (progressive enhancement):**

```bash
# Phase 1: Use CalDave's built-in email delivery (no configuration)
curl -s -X POST https://caldave.ai/calendars/cal_abc/events \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title": "Meeting", "start": "...", "end": "...", "attendees": ["bob@company.com"]}'
# => Invite sent from cal-abc@invite.caldave.ai

# Phase 2: Configure your own SMTP for branded emails
curl -s -X PUT https://caldave.ai/agents/smtp \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "host": "smtp.agentmail.to",
    "port": 465,
    "username": "inbox@agentmail.to",
    "password": "...",
    "from": "inbox@agentmail.to"
  }'

# Phase 3: Test it works
curl -s -X POST https://caldave.ai/agents/smtp/test \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"to": "test@example.com"}'
# => { "success": true, "message": "Test email sent successfully." }

# Phase 4: Remove it if you don't need it anymore
curl -s -X DELETE https://caldave.ai/agents/smtp \
  -H "Authorization: Bearer $API_KEY"
# => Reverts to CalDave built-in delivery
```

**Webhook example (progressive enhancement):**

```bash
# Phase 1: Poll for events (no webhook needed)
curl -s https://caldave.ai/calendars/cal_abc/upcoming \
  -H "Authorization: Bearer $API_KEY"

# Phase 2: Add webhooks when ready
curl -s -X PATCH https://caldave.ai/calendars/cal_abc \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"webhook_url": "https://my-agent.com/hooks", "webhook_secret": "s3cret"}'

# Phase 3: Verify webhook works
curl -s -X POST https://caldave.ai/calendars/cal_abc/webhook/test \
  -H "Authorization: Bearer $API_KEY"
```

---

## 20. Agent-First Ownership Model

**Files:** `src/routes/agents.js`, `src/routes/humans.js`

**Problem:** Traditional APIs are human-first: a human creates an account, generates API keys, and grants them to applications. This doesn't work when the agent needs to be autonomous.

**Pattern:** Agents are the primary citizens. They create themselves, manage their own calendars, and operate independently. Humans can optionally "claim" agents after the fact for oversight and key management.

**The inverted ownership flow:**

```bash
# 1. Agent creates itself (no human involved)
curl -s -X POST https://caldave.ai/agents \
  -H "Content-Type: application/json" \
  -d '{"name": "Meeting Bot"}'
# => { "agent_id": "agt_abc", "api_key": "sk_live_..." }

# 2. Agent creates calendars and events autonomously
curl -s -X POST https://caldave.ai/calendars \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{"name": "Team Calendar"}'

# 3. (Optional, later) Human creates an account
# Via browser: GET /signup

# 4. (Optional) Human claims the agent
curl -s -X POST https://caldave.ai/agents/claim \
  -H "X-Human-Key: hk_live_human_key_here" \
  -H "Content-Type: application/json" \
  -d '{"api_key": "sk_live_the_agents_key"}'
# => { "agent_id": "agt_abc", "claimed": true, "owned_by": "hum_xyz" }
```

The agent can also be auto-associated with a human at creation time:

```bash
curl -s -X POST https://caldave.ai/agents \
  -H "X-Human-Key: hk_live_human_key_here" \
  -H "Content-Type: application/json" \
  -d '{"name": "Meeting Bot"}'
# => { "agent_id": "agt_abc", "api_key": "sk_live_...", "owned_by": "hum_xyz" }
```

---

## 21. Rate Limiting with Standard Headers

**File:** `src/middleware/rateLimit.js`

**Problem:** Agents need to know rate limits programmatically to implement backoff. Custom rate limit headers or undocumented limits lead to hard-to-debug 429 errors.

**Pattern:** Use RFC draft-7 standard headers on every response. Different limits for different operations.

**Implementation:**

```javascript
const apiLimiter = rateLimit({
  windowMs: 60 * 1000,
  limit: 1000,
  standardHeaders: 'draft-7',    // RateLimit-Limit, RateLimit-Remaining, RateLimit-Reset
  legacyHeaders: false,           // No X-RateLimit-* headers
  message: { error: 'Too many requests, please try again later' },
});
```

**Rate limit tiers:**

| Operation | Limit | Window |
|---|---|---|
| General API | 1000 requests | per minute per IP |
| Agent creation | 20 requests | per hour per IP |
| Inbound email | 60 requests | per minute per IP |
| Human auth (login/signup) | 10 requests | per 15 minutes per IP |

**Headers on every response:**

```
RateLimit-Limit: 1000
RateLimit-Remaining: 997
RateLimit-Reset: 45
```

The `RateLimit-Reset` value is seconds until the window resets. An agent can read these headers on every response and back off before hitting the limit.

The 429 error response follows the consistent error format:

```json
{ "error": "Too many requests, please try again later" }
```

---

## 22. MCP Support

**File:** `src/index.js`, `src/routes/mcp.mjs`

**Problem:** HTTP APIs require agents to construct URLs, set headers, format JSON bodies, and parse responses. AI agents that support MCP (Model Context Protocol) can use structured tool calls instead, which are more natural for LLMs.

**Pattern:** Expose the same API capabilities through MCP tool calls, in addition to the HTTP API. Agents that support MCP can connect with just a URL and API key.

**Remote MCP (HTTP transport):**

The API exposes a Streamable HTTP endpoint at `/mcp`:

```json
{
  "url": "https://caldave.ai/mcp",
  "headers": {
    "Authorization": "Bearer sk_live_..."
  }
}
```

**Local MCP (STDIO transport):**

```bash
npx caldave-mcp
```

The MCP server includes:
- 24 tools covering all API endpoints
- A `caldave://guide` resource with a getting-started guide
- Structured instructions with quick-start workflow and tool selection guide

---

## 23. Welcome Event

**File:** `src/routes/calendars.js`

**Problem:** After creating a calendar, it's empty. An agent can't test event retrieval without first creating an event. This adds a step to the bootstrapping flow.

**Pattern:** New calendars automatically include a welcome event (scheduled 7 days out) so the agent can immediately test event retrieval. The welcome event can be disabled for production use.

```bash
# Default: welcome event created
curl -s -X POST https://caldave.ai/calendars \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Work"}'

# Production: skip the welcome event
curl -s -X POST https://caldave.ai/calendars \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Work", "welcome_event": false}'
```

This means immediately after creating a calendar, `GET /calendars/:id/upcoming` returns data instead of an empty array -- useful for agents that are testing their integration.

---

## 24. Pre-Computed URLs in Responses

**File:** `src/routes/calendars.js`

**Problem:** After creating a calendar, the agent needs to know the iCal feed URL (for subscribing from Google Calendar) and the inbound webhook URL (for receiving email invites). Constructing these URLs from component parts is error-prone.

**Pattern:** The calendar creation response includes fully assembled URLs that are ready to use.

```bash
curl -s -X POST https://caldave.ai/calendars \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Work", "timezone": "America/New_York"}'
```

Response:

```json
{
  "calendar_id": "cal_a1b2c3XyZ",
  "name": "Work",
  "timezone": "America/New_York",
  "email": "cal-a1b2c3XyZ@invite.caldave.ai",
  "ical_feed_url": "https://caldave.ai/feeds/cal_a1b2c3XyZ.ics?token=feed_abc123...",
  "inbound_webhook_url": "https://caldave.ai/inbound/inb_def456..."
}
```

The agent doesn't need to know the URL structure or token format. It just uses the URLs from the response.

---

## 25. iCal Feed with Caching Headers

**File:** `src/routes/feeds.js`

**Problem:** Calendar clients (Google Calendar, Apple Calendar) poll iCal feeds frequently. Without caching, every poll re-downloads the full calendar data.

**Pattern:** Token-based authentication via query parameter (no headers needed for calendar client subscriptions), ETag-based caching with `If-None-Match` support, and `Cache-Control` headers.

```bash
# First request -- returns full feed
curl -s "https://caldave.ai/feeds/cal_abc123.ics?token=feed_xyz..."
# Headers: ETag: "abc123", Cache-Control: public, max-age=300

# Subsequent request with ETag -- returns 304 if unchanged
curl -s "https://caldave.ai/feeds/cal_abc123.ics?token=feed_xyz..." \
  -H 'If-None-Match: "abc123"'
# => 304 Not Modified (no body)
```

The feed URL is designed to work with calendar apps that only support URL-based subscriptions (no custom headers).

---

## 26. Inbound Email Normalization

**File:** `src/routes/inbound.js`

**Problem:** Different email providers send webhooks in different formats. Postmark includes base64-encoded attachments inline. AgentMail provides attachment metadata that requires a separate API call to fetch content. An agent shouldn't need to handle multiple provider formats.

**Pattern:** A `normalizePayload()` function converts any supported provider's format into a common shape before processing.

```javascript
function normalizePayload(body) {
  if (isAgentMail(body)) {
    const msg = body.message;
    return {
      subject: msg.subject || '',
      textBody: msg.text || '',
      attachments: (msg.attachments || []).map((a) => ({
        ct: a.content_type || '',
        name: a.filename || '',
        content: null,    // fetched via API later
        agentmail: { inboxId: msg.inbox_id, messageId: msg.message_id, attachmentId: a.attachment_id },
      })),
    };
  }

  // Postmark format
  return {
    subject: body.Subject || '',
    textBody: body.TextBody || '',
    attachments: (body.Attachments || []).map((a) => ({
      ct: a.ContentType || '',
      name: a.Name || '',
      content: a.Content || null,   // base64 inline
    })),
  };
}
```

The shared `processInboundEmail()` function then works with the normalized shape regardless of which provider sent the webhook. Always returns 200 to prevent webhook retries, even on errors.

---

## 27. Fire-and-Forget Webhook Delivery

**File:** `src/lib/webhooks.js`

**Problem:** Webhook delivery shouldn't slow down API responses. If the agent's webhook receiver is down or slow, the API response to the original request shouldn't be delayed.

**Pattern:** Wrap webhook delivery in an async IIFE that catches and logs all errors. The route handler calls `fireWebhook()` after sending the response.

```javascript
function fireWebhook(calendarId, type, eventData) {
  (async () => {
    const { rows } = await pool.query(
      'SELECT webhook_url, webhook_secret FROM calendars WHERE id = $1',
      [calendarId]
    );
    if (rows.length === 0 || !rows[0].webhook_url) return;

    const payload = {
      type,
      calendar_id: calendarId,
      event_id: eventData.id || null,
      event: eventData,
      timestamp: new Date().toISOString(),
    };

    const body = JSON.stringify(payload);
    const headers = { 'Content-Type': 'application/json', 'User-Agent': 'CalDave-Webhook/1.0' };

    if (webhook_secret) {
      headers['X-CalDave-Signature'] = crypto
        .createHmac('sha256', webhook_secret)
        .update(body)
        .digest('hex');
    }

    await fetch(webhook_url, { method: 'POST', headers, body, signal: AbortSignal.timeout(10000) });
  })().catch(err => {
    console.error('[webhook] Delivery error:', err.message);
  });
}
```

Used in route handlers:

```javascript
// Response is sent FIRST, webhook fires after
res.status(201).json(response);
fireWebhook(req.params.id, 'event.created', response);
```

---

## 28. Recurring Event Abstraction

**File:** `src/lib/recurrence.js`

**Problem:** RFC 5545 recurrence rules (RRULEs) are complex. Agents would need to understand recurrence expansion to know what events exist next Tuesday. Storing just the rule without materialized instances means every query requires real-time expansion.

**Pattern:** The agent specifies an RRULE, and the system materializes individual event instances for a 90-day rolling window. The agent queries instances just like standalone events.

```bash
# Create a recurring event
curl -s -X POST https://caldave.ai/calendars/cal_abc/events \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Daily standup",
    "start": "2026-04-03T09:00:00-04:00",
    "end": "2026-04-03T09:15:00-04:00",
    "recurrence": "FREQ=DAILY;BYDAY=MO,TU,WE,TH,FR"
  }'
```

Response:

```json
{
  "id": "evt_parent123",
  "title": "Daily standup",
  "status": "recurring",
  "recurrence": "FREQ=DAILY;BYDAY=MO,TU,WE,TH,FR",
  "instances_created": 65
}
```

The agent can now query individual instances:

```bash
# List upcoming instances -- they look like regular events
curl -s https://caldave.ai/calendars/cal_abc/upcoming \
  -H "Authorization: Bearer $API_KEY"
```

**Deletion modes for recurring events:**

```bash
# Cancel just this instance
curl -s -X DELETE "https://caldave.ai/calendars/cal_abc/events/evt_instance456?mode=single" \
  -H "Authorization: Bearer $API_KEY"

# Cancel this and all future instances
curl -s -X DELETE "https://caldave.ai/calendars/cal_abc/events/evt_instance456?mode=future" \
  -H "Authorization: Bearer $API_KEY"

# Delete the entire series
curl -s -X DELETE "https://caldave.ai/calendars/cal_abc/events/evt_parent123?mode=all" \
  -H "Authorization: Bearer $API_KEY"
```

Safety limits are enforced: maximum 1000 instances per window, and SECONDLY/MINUTELY frequencies are rejected.

---

## Summary: Design Principles

These 28 patterns are organized around a set of core principles:

1. **Self-bootstrapping.** An agent can go from zero knowledge to full API usage by following the chain: hit any URL, get pointed to `/man`, follow progressive recommendations. No human setup required.

2. **Mistake tolerance.** Unknown fields are rejected with specific messages. Type-prefixed IDs prevent cross-entity confusion. The 404 handler points to documentation. Aliases are accepted for commonly confused field names.

3. **Token efficiency.** Topic filtering and guide mode reduce payload size. Pre-computed URLs eliminate the need to construct URLs. ISO 8601 durations are machine-parseable in one line of code.

4. **Proactive notifications.** Webhooks deliver events to agents without polling. Time-based webhooks handle "event is starting now" notifications. The changelog notifies agents of new features.

5. **Protocol abstraction.** Inbound email parsing, iCal generation, RSVP email sending, recurrence expansion, and HMAC signing are all handled server-side. Agents work with JSON; the server handles the rest.

6. **Progressive enhancement.** Agents start with zero configuration and add capabilities (SMTP, webhooks, human ownership) as needed. No feature requires upfront setup.

7. **Machine-parseable everything.** JSON responses, ISO 8601 durations, standard rate limit headers, consistent error format, type-prefixed IDs. Every response is designed to be parsed by code, not read by humans.
