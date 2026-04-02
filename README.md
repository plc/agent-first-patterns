# Agent-First API Design Patterns

A reference of design patterns for building APIs that AI agents can discover, learn, and use autonomously. Each pattern includes the problem it solves, how to implement it, and concrete examples.

These patterns were extracted from building [CalDave](https://caldave.ai), a calendar-as-a-service API for AI agents, but are applicable to any SaaS tool built for agent consumers.

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

```
GET /man
```

No authentication required. Returns:

```json
{
  "overview": "A brief description of what this API does.",
  "base_url": "https://api.example.com",
  "available_topics": ["agents", "resources", "notifications", "errors"],
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
  "your_context": {
    "authenticated": false,
    "agent_id": null,
    "resources": []
  },
  "recommended_next_step": { "..." },
  "endpoints": [
    {
      "method": "POST",
      "path": "/agents",
      "description": "Create a new agent identity...",
      "auth": "none",
      "parameters": [
        { "name": "name", "in": "body", "required": false, "type": "string", "description": "Agent display name" }
      ],
      "example_curl": "curl -s -X POST \"https://api.example.com/agents\" ...",
      "example_response": { "..." }
    }
  ]
}
```

Every endpoint entry includes: method, path, description, auth requirement, parameter list (with name, location, type, required flag, description), a generated curl command, and an example response.

The endpoint catalog can be pre-computed at module load time since it is static:

```javascript
const CACHED_ENDPOINTS = getEndpoints();
```

```bash
# Unauthenticated -- discover the full API
curl -s https://api.example.com/man | jq '.endpoints | length'

# Authenticated -- get personalized context and recommendations
curl -s https://api.example.com/man \
  -H "Authorization: Bearer $API_KEY"
```

---

## 2. Progressive Recommendation Engine

**Problem:** An agent that reads 30 endpoints doesn't know which one to call first. Without guidance, agents make random or incorrect choices about what to do next.

**Pattern:** Inspect the agent's current state and return the single most important next action with a ready-to-execute curl command. The recommendation changes as the agent progresses through setup.

A recommendation chain for a generic SaaS API might look like:

| Agent State | Recommendation | Endpoint |
|---|---|---|
| Not authenticated | "Create an agent" | `POST /agents` |
| No name set | "Name your agent" | `PATCH /agents` |
| No resources created | "Create your first resource" | `POST /resources` |
| Resources but no activity | "Use the resource" | `POST /resources/:id/actions` |
| Not claimed by a human | "Claim this agent" | `POST /agents/claim` |
| Everything done | "Check for new activity" | `GET /resources/:id/activity` |

The recommendation always includes:
- `action` -- what to do (human-readable)
- `description` -- why this is the recommended next step
- `endpoint` -- which endpoint to call
- `curl` -- a ready-to-paste command with the agent's real IDs interpolated

```javascript
function buildRecommendation(context, apiKey) {
  if (!context.authenticated) {
    return {
      action: 'Create an agent',
      description: 'You are not authenticated. Create an agent to get an API key.',
      endpoint: 'POST /agents',
      curl: buildCurl('POST', '/agents', {
        body: { name: 'My Agent', description: 'What this agent does' },
      }),
    };
  }

  if (!context.agent_name) {
    return {
      action: 'Name your agent',
      description: 'Setting a name helps identify your agent in logs and notifications.',
      endpoint: 'PATCH /agents',
      curl: buildCurl('PATCH', '/agents', { apiKey, body: { name: 'My Agent' } }),
    };
  }

  if (context.resources.length === 0) {
    return {
      action: 'Create your first resource',
      description: 'You have an agent but no resources yet.',
      endpoint: 'POST /resources',
      curl: buildCurl('POST', '/resources', { apiKey, body: { name: 'My Resource' } }),
    };
  }

  // ...continue down the chain
}
```

The agent sees a different recommendation each time it calls `/man`, tracking its progress through the onboarding flow.

---

## 3. Soft Authentication

**Problem:** Discovery endpoints need to work without authentication (so new agents can find out how to create credentials), but authenticated agents should get personalized data. Standard auth middleware returns 401 for missing tokens, blocking unauthenticated access entirely.

**Pattern:** "Soft auth" resolves a Bearer token when present but never returns 401. If the token is valid, `req.agent` is populated. If missing or invalid, `req.agent` is set to `null` and the request proceeds.

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

Use soft auth on discovery endpoints (`/man`, `/changelog`). Use hard auth (returning 401) on resource CRUD endpoints. Use no auth on agent creation.

| Endpoint type | Auth style | Rationale |
|---|---|---|
| Discovery (`/man`, `/changelog`) | Soft auth | Must be accessible to unauthenticated agents |
| Resource CRUD | Hard auth (401) | Data must be scoped to the authenticated agent |
| Agent creation (`POST /agents`) | No auth | This is how agents get credentials |
| Webhooks/feeds with URL tokens | Token in URL | Third-party services can't set Bearer headers |

---

## 4. Zero-Auth Agent Bootstrapping

**Problem:** Traditional APIs require a human to create an account, log in, navigate to a dashboard, generate an API key, and configure it. AI agents can't do any of that.

**Pattern:** A single unauthenticated POST creates an agent identity and returns a usable API key. The agent goes from zero credentials to fully authenticated in one HTTP call.

```bash
curl -s -X POST https://api.example.com/agents \
  -H "Content-Type: application/json" \
  -d '{"name": "My Agent", "description": "Manages inventory for the warehouse"}'
```

Response:

```json
{
  "agent_id": "agt_x7y8z9AbCd",
  "api_key": "key_live_...",
  "name": "My Agent",
  "description": "Manages inventory for the warehouse",
  "message": "Store these credentials securely. The API key will not be shown again."
}
```

Key details:
- The API key is shown exactly once. Only the SHA-256 hash is stored.
- The warning message is itself agent-friendly: it tells the agent exactly what to do.
- `name` and `description` are optional, so the minimum bootstrapping call is just `POST /agents` with an empty body.
- Rate-limit agent creation (e.g. 20/hour per IP) to prevent abuse.

**Complete bootstrapping sequence in 3 calls:**

```bash
# 1. Create agent
RESPONSE=$(curl -s -X POST https://api.example.com/agents \
  -H "Content-Type: application/json" \
  -d '{"name": "My Agent"}')
API_KEY=$(echo $RESPONSE | jq -r '.api_key')

# 2. Create a resource
curl -s -X POST https://api.example.com/projects \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Q2 Campaign"}'

# 3. Start using it
curl -s -X POST https://api.example.com/projects/prj_abc123/tasks \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title": "Draft copy", "due": "2026-04-10T17:00:00Z"}'
```

---

## 5. Context Window Management

**Problem:** AI agents have limited context windows. The full `/man` response with 30+ endpoints can be thousands of tokens. An agent working on one feature doesn't need to see endpoints for unrelated features.

**Pattern:** Two mechanisms to reduce payload size:

**Topic filtering** (`?topic=`): Returns only endpoints matching the specified categories. Discovery endpoints are always included.

```bash
# Only see task-related endpoints
curl -s "https://api.example.com/man?topic=tasks" \
  -H "Authorization: Bearer $API_KEY" | jq '.endpoints | length'
# => 6 (vs 30+ for the full catalog)

# Multiple topics
curl -s "https://api.example.com/man?topic=tasks,notifications"

# Invalid topic returns a helpful error
curl -s "https://api.example.com/man?topic=bogus"
# => { "error": "Unknown topic: bogus. Available: agents, projects, tasks, notifications, errors" }
```

**Guide mode** (`?guide`): Returns only the overview, agent context, and recommended next step. No endpoint catalog at all.

```bash
curl -s "https://api.example.com/man?guide" \
  -H "Authorization: Bearer $API_KEY"
```

Response (much smaller):

```json
{
  "overview": "A brief description of what this API does.",
  "base_url": "https://api.example.com",
  "rate_limits": { "api": "1000/min", "agent_creation": "20/hour" },
  "error_format": { "..." },
  "your_context": {
    "authenticated": true,
    "agent_id": "agt_abc123",
    "projects": [{ "id": "prj_xyz", "name": "Q2 Campaign", "task_count": 12 }]
  },
  "recommended_next_step": { "..." },
  "discover_more": {
    "full_api_reference": "GET /man (without ?guide) returns all endpoints.",
    "changelog": "GET /changelog shows new features since you signed up."
  }
}
```

---

## 6. Generated Examples with Real Credentials

**Problem:** Example code in docs typically uses placeholder values like `YOUR_API_KEY` and `YOUR_PROJECT_ID`. An agent has to figure out what to substitute. This is error-prone and wastes tokens.

**Pattern:** When the agent is authenticated, every curl example in the `/man` response is generated with the agent's actual API key and real resource IDs interpolated.

```javascript
function buildCurl(method, path, { apiKey, resourceId, body } = {}) {
  let resolved = path.replace(':id', resourceId || 'RESOURCE_ID');
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

When unauthenticated, examples use placeholder values. When authenticated, they use the real values. The agent can copy any curl command from the `/man` response and execute it directly without substitution.

---

## 7. Agent-Aware Changelog

**Problem:** Agents don't know when new features are added. They continue using the same endpoints they were originally programmed with, missing improvements.

**Pattern:** A structured changelog endpoint that, when authenticated, splits entries into "changes since your agent was created" vs "changes before you existed." Includes personalized recommendations and explicit polling guidance.

```bash
curl -s https://api.example.com/changelog \
  -H "Authorization: Bearer $API_KEY"
```

Authenticated response:

```json
{
  "description": "API changelog. Lists new features, improvements, and fixes.",
  "poll_recommendation": "Check this endpoint approximately once per week.",
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
          "title": "Batch operations endpoint",
          "description": "New POST /batch endpoint for submitting multiple operations...",
          "endpoints": ["POST /batch"],
          "docs": "https://api.example.com/docs#batch"
        }
      ]
    }
  ],
  "changes_since_signup_count": 3,
  "changelog": [ "...older entries..." ],
  "recommendations": [
    {
      "action": "Try batch operations",
      "why": "You make many individual API calls that could be batched.",
      "how": "POST /batch with an array of operations.",
      "docs": "https://api.example.com/docs#batch"
    }
  ]
}
```

Unauthenticated response (simpler):

```json
{
  "description": "API changelog...",
  "poll_recommendation": "Check this endpoint approximately once per week.",
  "tip": "Pass your API key as a Bearer token to see which changes are new since your agent was created.",
  "changelog": [ "...all entries..." ]
}
```

Each changelog entry has a `type` field (`feature`, `improvement`, `fix`) so agents can programmatically filter for what matters.

---

## 8. Unknown Field Rejection

**Problem:** LLMs frequently hallucinate field names. If `due_date` is the internal column name but the API accepts `due`, an agent might send `due_date` and the API silently ignores it. The agent thinks it set the due date, but nothing happened.

**Pattern:** Every write endpoint validates incoming fields against a known set and returns a specific error listing the unknown fields. No field is ever silently ignored.

```javascript
const KNOWN_FIELDS = new Set(['title', 'description', 'due', 'status', 'priority', 'metadata']);

function checkUnknownFields(body, knownFields) {
  const unknown = Object.keys(body).filter(k => !knownFields.has(k));
  if (unknown.length === 0) return null;
  return `Unknown field${unknown.length > 1 ? 's' : ''}: ${unknown.join(', ')}`;
}

// In route handler:
const unknownErr = checkUnknownFields(req.body, KNOWN_FIELDS);
if (unknownErr) return res.status(400).json({ error: unknownErr });
```

Example of the error an agent receives:

```bash
curl -s -X POST https://api.example.com/projects/prj_abc/tasks \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title": "Test", "due_date": "2026-04-01T10:00:00Z", "assignee": "bob"}'
```

Response (400):

```json
{
  "error": "Unknown fields: due_date, assignee"
}
```

The agent immediately knows `due_date` and `assignee` aren't valid. It can check the `/man` endpoint to find the correct names.

Apply this pattern on every write endpoint in the API, not just some.

---

## 9. Type-Prefixed IDs

**Problem:** APIs that use UUIDs or numeric IDs make it easy to pass a project ID where a task ID is expected. The error message ("not found") doesn't tell the agent it used the wrong type of ID.

**Pattern:** Every ID includes a prefix that identifies its entity type. An agent (or a human reviewing logs) can immediately tell what kind of entity an ID refers to.

```javascript
const { customAlphabet } = require('nanoid');

const alphabet = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz';
const shortId = customAlphabet(alphabet, 12);
const longId = customAlphabet(alphabet, 32);

const agentId    = () => `agt_${shortId()}`;     // agt_x7y8z9AbCd
const projectId  = () => `prj_${shortId()}`;     // prj_a1b2c3XyZ
const taskId     = () => `tsk_${shortId()}`;     // tsk_p4q5r6StUv
const apiKey     = () => `key_live_${longId()}`; // key_live_abc123...
```

Design decisions:
- **Short IDs** (12 random chars) for entities that appear in URLs and logs
- **Long IDs** (32 random chars) for secrets and tokens
- **Alphanumeric alphabet** (no special chars) so IDs are URL-safe without encoding
- **Consistent prefix convention** -- agents learn the pattern and can self-check before sending requests

---

## 10. Consistent Error Format

**Problem:** Inconsistent error formats force agents to handle multiple shapes. Some APIs return `{ "message": "..." }`, others `{ "errors": [...] }`, others `{ "code": 123, "detail": "..." }`.

**Pattern:** Every error response in the entire API uses the same shape: `{ "error": "Human-readable message" }`. Document this in the `/man` response so agents know what to expect before they encounter errors.

```json
// 400 - Validation error
{ "error": "title is required" }

// 400 - Unknown fields
{ "error": "Unknown fields: due_date, assignee" }

// 400 - Type error
{ "error": "name must be a string" }

// 400 - Size limit
{ "error": "description exceeds 64KB limit" }

// 401 - Auth error
{ "error": "Missing or invalid Authorization header" }

// 404 - Not found
{ "error": "Project not found" }

// 404 - Global catch-all
{ "error": "Not found. Try GET /man for the API reference." }

// 429 - Rate limit
{ "error": "Too many requests, please try again later" }

// 500 - Server error
{ "error": "Failed to create task" }
```

The error message is always a single string. Agent code to handle any error:

```javascript
const data = await response.json();
if (data.error) {
  console.log("API error:", data.error);
}
```

---

## 11. 404 as Discovery Mechanism

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

**Problem:** When an agent triggers a server error (500), traditional APIs just say "Internal server error." The agent can't debug the issue, and a human has to check server logs.

**Pattern:** Log all server errors to a database table with the agent's ID attached. Expose a `GET /errors` endpoint so agents can query their own errors.

Error logging utility:

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

Call it from every catch block:

```javascript
catch (err) {
  logError(err, { route: 'POST /projects/:id/tasks', method: 'POST', agent_id: req.agent?.id });
  res.status(500).json({ error: 'Failed to create task' });
}
```

Agents debug their own errors:

```bash
# List recent errors
curl -s https://api.example.com/errors \
  -H "Authorization: Bearer $API_KEY"

# Response:
{
  "errors": [
    {
      "id": 42,
      "route": "POST /projects/:id/tasks",
      "method": "POST",
      "status_code": 500,
      "message": "invalid input syntax for type timestamp...",
      "agent_id": "agt_abc123",
      "created_at": "2026-04-01T15:30:00.000Z"
    }
  ],
  "count": 1
}

# Get full stack trace
curl -s https://api.example.com/errors/42 \
  -H "Authorization: Bearer $API_KEY"

# Filter by route
curl -s "https://api.example.com/errors?route=tasks" \
  -H "Authorization: Bearer $API_KEY"
```

Errors are scoped: an agent can only see errors associated with its own `agent_id`.

---

## 13. Polling Hints

**Problem:** An agent that polls on a fixed interval wastes API calls when nothing is happening soon, and misses things when timing is tight.

**Pattern:** Include machine-readable hints that tell the agent when to poll next.

**Duration-based hint:** Return an ISO 8601 duration indicating when the next relevant item occurs.

```json
{
  "items": [ "..." ],
  "next_item_in": "PT14M30S"
}
```

`PT14M30S` means 14 minutes and 30 seconds. The agent schedules its next poll accordingly instead of using a fixed interval. Parseable by standard libraries in any language.

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

**Frequency-based hint:** For low-frequency endpoints like changelogs, include explicit guidance:

```json
{
  "poll_recommendation": "Check this endpoint approximately once per week."
}
```

---

## 14. Webhook System

**Problem:** Polling wastes resources and introduces latency. Agents need to know immediately when things change.

**Pattern:** Support two categories of webhooks:

**Mutation webhooks** fire immediately from route handlers when data changes:
- `resource.created`
- `resource.updated`
- `resource.deleted`

**Time-based webhooks** fire from a background poller when a scheduled time arrives:
- `task.due` -- a task's due date has arrived
- `reminder.triggered` -- a scheduled reminder fires

**Setting up webhooks on a resource:**

```bash
curl -s -X POST https://api.example.com/projects \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Q2 Campaign",
    "webhook_url": "https://my-agent.example.com/webhooks",
    "webhook_secret": "my-hmac-secret-123"
  }'
```

**Webhook test endpoint** so agents can verify their receiver works before relying on it:

```bash
curl -s -X POST https://api.example.com/projects/prj_abc/webhook/test \
  -H "Authorization: Bearer $API_KEY"
# => { "success": true, "status_code": 200, "message": "Webhook delivered successfully." }
```

**Webhook payload format:**

```json
{
  "type": "resource.created",
  "project_id": "prj_abc123",
  "resource_id": "tsk_xyz789",
  "resource": {
    "id": "tsk_xyz789",
    "title": "Draft copy",
    "status": "open"
  },
  "timestamp": "2026-04-02T18:30:00.000Z"
}
```

**HMAC-SHA256 signing** when `webhook_secret` is set:

```javascript
const crypto = require('crypto');

// Signing (server side)
const signature = crypto
  .createHmac('sha256', webhook_secret)
  .update(rawBody)
  .digest('hex');
headers['X-Signature'] = signature;

// Verification (agent side)
function verifyWebhook(body, signature, secret) {
  const expected = crypto
    .createHmac('sha256', secret)
    .update(body)  // raw body string, not re-serialized JSON
    .digest('hex');
  return crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected));
}
```

---

## 15. Field Alias Acceptance

**Problem:** LLMs hallucinate field names based on training data. If the more common term in public codebases differs from your API's field name, agents will consistently get it wrong.

**Pattern:** Accept common aliases for fields that agents are likely to confuse. Normalize silently before validation.

```javascript
function normalizeBody(body) {
  // Accept "due_date" as alias for "due"
  if (body.due_date !== undefined && body.due === undefined) {
    body.due = body.due_date;
  }
  delete body.due_date;

  // Accept "desc" as alias for "description"
  if (body.desc !== undefined && body.description === undefined) {
    body.description = body.desc;
  }
  delete body.desc;
}
```

Include the aliases in the known fields set so they don't trigger the unknown field rejection (pattern #8). The canonical field takes priority when both are present.

---

## 16. Arbitrary Metadata Field

**Problem:** Different agents need to store different auxiliary data on resources. One agent might track an external CRM ID, another might store processing state, another might attach tags. A rigid schema can't anticipate every agent's needs.

**Pattern:** Resources have a `metadata` JSONB field that accepts arbitrary structured data up to a size limit. The API stores it as-is and returns it as-is.

```bash
curl -s -X POST https://api.example.com/projects/prj_abc/tasks \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Follow up with client",
    "due": "2026-04-10T17:00:00Z",
    "metadata": {
      "crm_deal_id": "deal_789",
      "source": "email_triage",
      "priority_score": 0.85,
      "tags": ["client", "urgent"]
    }
  }'
```

The metadata is returned in all responses for that resource. Set a reasonable size limit (e.g. 16KB) and validate it's an object, not an array or primitive.

---

## 17. Plain Text Rendering

**Problem:** Some agents operate in text-only environments (terminals, chat interfaces, simple HTTP clients). Parsing JSON to produce a human-readable view adds unnecessary complexity.

**Pattern:** Offer a `text/plain` endpoint that returns pre-formatted output that works directly with `curl`.

```bash
curl -s https://api.example.com/projects/prj_abc/view \
  -H "Authorization: Bearer $API_KEY"
```

Response (`text/plain`):

```
Project: Q2 Campaign

Status    Title                  Due          Assignee
--------- ---------------------- ------------ -----------
open      Draft copy             Apr 10       (unassigned)
open      Design review          Apr 12       (unassigned)
done      Competitor research    Apr 03       (unassigned)
```

This is useful when an agent needs to include a snapshot in a text-based report, log, or chat message without parsing JSON.

---

## 18. Protocol Abstraction

**Problem:** Interoperating with external systems often requires understanding complex protocols (email/SMTP, iCal, OAuth handshakes, file format parsing). Agents shouldn't need to implement protocol-level details.

**Pattern:** The API handles all protocol complexity server-side. The agent works with simple JSON fields; the API translates to/from the underlying protocols.

For example, if your API receives data from external services via webhooks (email providers, payment processors, form builders), normalize all provider formats into a common shape before exposing them to agents:

```javascript
function normalizePayload(body) {
  if (isProviderA(body)) {
    return {
      subject: body.message.subject || '',
      content: body.message.text || '',
      attachments: (body.message.attachments || []).map(normalize),
    };
  }
  // Provider B has a different format
  return {
    subject: body.Subject || '',
    content: body.TextBody || '',
    attachments: (body.Attachments || []).map(normalize),
  };
}
```

The agent sees a clean JSON resource. It never has to know which external provider originated the data, what wire format it arrived in, or what protocol was used to respond.

Similarly, if your API sends outbound communications (emails, SMS, Slack messages), the agent provides simple parameters (recipient, message) and the API handles protocol details (SMTP, message formatting, delivery confirmation):

```bash
# Agent sends a simple JSON request
curl -s -X POST https://api.example.com/projects/prj_abc/notify \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"to": "team@company.com", "message": "Sprint review moved to Friday"}'

# Response confirms delivery without exposing protocol details
{ "sent": true, "message_id": "msg_abc123" }
```

---

## 19. Progressive Enhancement

**Problem:** Forcing agents to configure everything upfront creates a high barrier to entry. Many features might never be needed.

**Pattern:** Start with sensible defaults that work out of the box. Let agents add capabilities incrementally as they need them.

```bash
# Phase 1: Core functionality works with zero configuration
curl -s -X POST https://api.example.com/projects/prj_abc/tasks \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title": "Draft copy", "due": "2026-04-10T17:00:00Z"}'
# => Works. Notifications use built-in delivery.

# Phase 2: Add webhooks when the agent is ready for real-time updates
curl -s -X PATCH https://api.example.com/projects/prj_abc \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"webhook_url": "https://my-agent.com/hooks", "webhook_secret": "s3cret"}'

# Phase 3: Verify webhook works before relying on it
curl -s -X POST https://api.example.com/projects/prj_abc/webhook/test \
  -H "Authorization: Bearer $API_KEY"

# Phase 4: Configure custom outbound delivery (e.g. own SMTP, Slack bot)
curl -s -X PUT https://api.example.com/agents/integrations/smtp \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"host": "smtp.example.com", "port": 465, "username": "...", "password": "...", "from": "agent@example.com"}'

# Phase 5: Test the integration
curl -s -X POST https://api.example.com/agents/integrations/smtp/test \
  -H "Authorization: Bearer $API_KEY"
# => { "success": true, "message": "Test email sent successfully." }

# Phase 6: Remove it if not needed -- reverts to built-in defaults
curl -s -X DELETE https://api.example.com/agents/integrations/smtp \
  -H "Authorization: Bearer $API_KEY"
```

The key principle: everything works at phase 1. Each subsequent phase adds capability but is never required.

---

## 20. Agent-First Ownership Model

**Problem:** Traditional APIs are human-first: a human creates an account, generates API keys, and grants them to applications. This doesn't work when the agent needs to be autonomous.

**Pattern:** Agents are the primary citizens. They create themselves, manage their own resources, and operate independently. Humans can optionally "claim" agents after the fact for oversight and key management.

```bash
# 1. Agent creates itself (no human involved)
curl -s -X POST https://api.example.com/agents \
  -H "Content-Type: application/json" \
  -d '{"name": "Inventory Bot"}'
# => { "agent_id": "agt_abc", "api_key": "key_live_..." }

# 2. Agent operates autonomously
curl -s -X POST https://api.example.com/projects \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Warehouse Tracking"}'

# 3. (Optional, later) Human creates an account via browser
# GET /signup

# 4. (Optional) Human claims the agent for oversight
curl -s -X POST https://api.example.com/agents/claim \
  -H "X-Human-Key: hk_live_human_key_here" \
  -H "Content-Type: application/json" \
  -d '{"api_key": "key_live_the_agents_key"}'
# => { "agent_id": "agt_abc", "claimed": true, "owned_by": "hum_xyz" }
```

Agents can also be auto-associated with a human at creation time via an optional header:

```bash
curl -s -X POST https://api.example.com/agents \
  -H "X-Human-Key: hk_live_human_key_here" \
  -H "Content-Type: application/json" \
  -d '{"name": "Inventory Bot"}'
# => { "agent_id": "agt_abc", "api_key": "key_live_...", "owned_by": "hum_xyz" }
```

This inverts the traditional model. The agent is the primary entity. Human oversight is optional and additive.

---

## 21. Rate Limiting with Standard Headers

**Problem:** Agents need to know rate limits programmatically to implement backoff. Custom rate limit headers or undocumented limits lead to hard-to-debug 429 errors.

**Pattern:** Use RFC draft-7 standard headers on every response. Apply different limits to different operation types.

```javascript
const apiLimiter = rateLimit({
  windowMs: 60 * 1000,
  limit: 1000,
  standardHeaders: 'draft-7',    // RateLimit-Limit, RateLimit-Remaining, RateLimit-Reset
  legacyHeaders: false,           // No X-RateLimit-* headers
  message: { error: 'Too many requests, please try again later' },
});
```

Headers on every response:

```
RateLimit-Limit: 1000
RateLimit-Remaining: 997
RateLimit-Reset: 45
```

`RateLimit-Reset` is seconds until the window resets. An agent can read these on every response and back off before hitting the limit.

Apply different tiers for different operations:

| Operation | Limit | Window |
|---|---|---|
| General API | 1000 requests | per minute per IP |
| Agent creation | 20 requests | per hour per IP |
| Inbound webhooks | 60 requests | per minute per IP |
| Human auth | 10 requests | per 15 minutes per IP |

---

## 22. MCP Support

**Problem:** HTTP APIs require agents to construct URLs, set headers, format JSON bodies, and parse responses. AI agents that support MCP (Model Context Protocol) can use structured tool calls instead, which is more natural for LLMs.

**Pattern:** Expose the same API capabilities through MCP tool calls, in addition to the HTTP API. Offer both remote (HTTP transport) and local (STDIO) modes.

Remote MCP (Streamable HTTP):

```json
{
  "url": "https://api.example.com/mcp",
  "headers": {
    "Authorization": "Bearer key_live_..."
  }
}
```

Local MCP (STDIO):

```bash
npx your-mcp-package
```

Include in the MCP server:
- Tools covering all API endpoints
- A guide resource (`yourapp://guide`) with getting-started content
- Structured instructions with workflow descriptions and tool selection guidance

MCP is additive. The HTTP API remains the primary interface. MCP provides an alternative integration path for agents that support it.

---

## 23. Seed Data on Resource Creation

**Problem:** After creating a new container resource (a project, workspace, folder), it's empty. An agent can't test retrieval without first creating child resources. This adds an extra step to the bootstrapping flow.

**Pattern:** Automatically create a sample resource inside new containers so the agent can immediately test listing and retrieval. Allow opting out for production use.

```bash
# Default: sample item created
curl -s -X POST https://api.example.com/projects \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Q2 Campaign"}'
# => Project created with a sample task already inside it

# Production: skip sample data
curl -s -X POST https://api.example.com/projects \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Q2 Campaign", "seed_data": false}'
```

Immediately after creation, `GET /projects/:id/tasks` returns data instead of an empty array. This is useful for agents testing their integration and avoids a "chicken and egg" problem during bootstrapping.

---

## 24. Pre-Computed URLs in Responses

**Problem:** After creating a resource, the agent needs to know derived URLs (feed URLs, webhook URLs, share links). Constructing these from component parts is error-prone.

**Pattern:** Resource creation responses include fully assembled URLs that are ready to use.

```bash
curl -s -X POST https://api.example.com/projects \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Q2 Campaign"}'
```

Response:

```json
{
  "id": "prj_a1b2c3XyZ",
  "name": "Q2 Campaign",
  "feed_url": "https://api.example.com/feeds/prj_a1b2c3XyZ.json?token=feed_abc123...",
  "webhook_inbound_url": "https://api.example.com/inbound/inb_def456...",
  "share_url": "https://app.example.com/projects/prj_a1b2c3XyZ"
}
```

The agent doesn't need to know URL structures or token formats. It just uses the URLs directly from the response.

---

## 25. Fire-and-Forget Webhook Delivery

**Problem:** Webhook delivery shouldn't slow down API responses. If the agent's webhook receiver is down or slow, the API response to the original request shouldn't be delayed.

**Pattern:** Wrap webhook delivery in an async IIFE that catches and logs all errors. Call it after the response is sent.

```javascript
function fireWebhook(resourceId, type, data) {
  (async () => {
    const { rows } = await pool.query(
      'SELECT webhook_url, webhook_secret FROM projects WHERE id = $1',
      [resourceId]
    );
    if (rows.length === 0 || !rows[0].webhook_url) return;

    const { webhook_url, webhook_secret } = rows[0];
    const payload = {
      type,
      resource_id: data.id || null,
      resource: data,
      timestamp: new Date().toISOString(),
    };

    const body = JSON.stringify(payload);
    const headers = { 'Content-Type': 'application/json', 'User-Agent': 'YourApp-Webhook/1.0' };

    if (webhook_secret) {
      headers['X-Signature'] = crypto
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
fireWebhook(req.params.id, 'resource.created', response);
```

The 10-second timeout prevents a single slow receiver from accumulating open connections.

---

## Summary: Design Principles

These 25 patterns are organized around 7 core principles:

1. **Self-bootstrapping.** An agent can go from zero knowledge to full API usage by following the chain: hit any URL, get pointed to `/man`, follow progressive recommendations. No human setup required.

2. **Mistake tolerance.** Unknown fields are rejected with specific messages. Type-prefixed IDs prevent cross-entity confusion. The 404 handler points to documentation. Aliases are accepted for commonly confused field names.

3. **Token efficiency.** Topic filtering and guide mode reduce payload size. Pre-computed URLs eliminate the need to construct URLs. ISO 8601 durations are machine-parseable in a single line of code.

4. **Proactive notifications.** Webhooks deliver changes to agents without polling. Time-based webhooks handle scheduled triggers. The changelog notifies agents of new features.

5. **Protocol abstraction.** External protocol complexity (email, file formats, auth handshakes) is handled server-side. Agents work with JSON; the server handles the rest.

6. **Progressive enhancement.** Agents start with zero configuration and add capabilities (custom delivery, webhooks, human ownership) as needed. No feature requires upfront setup.

7. **Machine-parseable everything.** JSON responses, ISO 8601 durations, standard rate limit headers, consistent error format, type-prefixed IDs. Every response is designed to be parsed by code, not read by humans.
