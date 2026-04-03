# Agent-First API Design Patterns

A reference of design patterns for building APIs that AI agents can discover, learn, and use autonomously.

Each pattern describes a problem, the solution, trade-offs, and when not to use it. Examples use generic placeholder domains (`api.example.com`). For a production implementation of every pattern, see [CalDave: Agent-First Patterns in Practice](CALDAVE.md).

---

## Table of Contents

- [If You Implement Nothing Else](#if-you-implement-nothing-else)
- [Design Principles](#design-principles)
- [Patterns](#patterns)
  - [Essential Tier](#essential-tier)
    - [1. Self-Describing API Manual](#1-self-describing-api-manual)
    - [2. Zero-Auth Agent Bootstrapping](#2-zero-auth-agent-bootstrapping)
    - [3. Structured Error Handling](#3-structured-error-handling)
    - [4. Type-Prefixed IDs](#4-type-prefixed-ids)
    - [5. Idempotency Keys](#5-idempotency-keys)
  - [Important Tier](#important-tier)
    - [6. Agent-First Ownership Model](#6-agent-first-ownership-model)
    - [7. Progressive Enhancement](#7-progressive-enhancement)
    - [8. Pre-Computed URLs in Responses](#8-pre-computed-urls-in-responses)
    - [9. OpenAPI / Structured Tool Schemas](#9-openapi--structured-tool-schemas)
    - [10. Rate Limiting with Standard Headers](#10-rate-limiting-with-standard-headers)
    - [11. Pagination](#11-pagination)
    - [12. Protocol Abstraction](#12-protocol-abstraction)
  - [Nice-to-Have Tier](#nice-to-have-tier)
    - [13. Webhook System](#13-webhook-system)
    - [14. Polling Hints](#14-polling-hints)
    - [15. Batch Operations](#15-batch-operations)
    - [16. Field Alias Acceptance](#16-field-alias-acceptance)
    - [17. Arbitrary Metadata Field](#17-arbitrary-metadata-field)
    - [18. Agent-Aware Changelog](#18-agent-aware-changelog)
    - [19. Plain Text Rendering](#19-plain-text-rendering)
    - [20. MCP Support](#20-mcp-support)
- [About](#about)

---

## If You Implement Nothing Else

If you're building an agent-first API and can only implement a handful of patterns, start with these five. They provide the foundation for agent autonomy and will deliver the most value with the least effort.

**Self-Describing API Manual** is the cornerstone. Without it, agents must rely on external documentation or hardcoded knowledge. A single JSON endpoint that describes all available operations lets agents discover and use your API programmatically.

**Zero-Auth Agent Bootstrapping** removes the human from the critical path. Traditional signup flows require dashboard navigation, email verification, and manual key generation. A single unauthenticated POST that returns a working API key means agents can provision themselves in seconds.

**Structured Error Handling** prevents silent failures and hallucination loops. When agents receive consistent, machine-parseable errors that explicitly list what went wrong, they can self-correct instead of retrying the same mistake indefinitely.

**Type-Prefixed IDs** eliminate an entire class of bugs where agents pass the wrong entity type to an endpoint. When every ID clearly identifies what it represents, both agents and humans debugging logs can spot mismatches immediately.

**Idempotency Keys** make your API safe for retries. Agents lose context, conversations get interrupted, and networks fail. Without idempotency, every retry creates duplicate resources. With it, agents can safely retry any operation.

These five patterns work together to create an API that agents can discover, authenticate with, use correctly, debug when things go wrong, and retry safely when things fail. Everything else builds on this foundation.

---

## Design Principles

These seven principles inform all the patterns in this document.

**1. Self-bootstrapping.** An agent should be able to go from zero knowledge to full API usage programmatically. No human setup, no external documentation, no hardcoded knowledge. The API itself provides everything the agent needs to get started.

**2. Mistake tolerance.** Agents will make mistakes. Hallucinated field names, wrong entity types, malformed requests. The API should catch these errors early, provide specific actionable feedback, and never silently ignore problems.

**3. Token efficiency.** Every byte in a response costs tokens. Large payloads waste context window space and increase latency. APIs should support filtering, provide only what's needed, and use compact machine-readable formats.

**4. Proactive notifications.** Polling wastes resources and introduces latency. When possible, push changes to agents via webhooks or other notification mechanisms. When polling is necessary, tell agents when to poll next.

**5. Protocol abstraction.** Agents shouldn't implement SMTP, parse iCal files, or handle OAuth handshakes. Complex protocol details belong server-side. Agents send and receive JSON; the API handles the rest.

**6. Progressive enhancement.** Don't force agents to configure everything upfront. Start with working defaults. Let agents add capabilities incrementally as they discover they need them. No feature should block basic usage.

**7. Machine-parseable everything.** Every response should be designed for programmatic consumption. Structured errors, standard headers, consistent ID formats, machine-readable time formats. If a human has to parse it, an agent will struggle.

---

## Patterns

The patterns are organized into three tiers based on impact and implementation complexity.

**Essential** patterns provide foundational capabilities that most agent-first APIs need. They have high impact and should be prioritized.

**Important** patterns address common needs and significantly improve the agent experience. They require more effort but deliver clear value.

**Nice-to-Have** patterns solve specific problems or support advanced use cases. Implement these based on your specific requirements.

---

## Essential Tier

### 1. Self-Describing API Manual

**Tier:** Essential

**Problem:** AI agents have no prior knowledge of your API. Traditional documentation is HTML designed for human eyes. Agents need to discover capabilities, understand parameters, and start making calls programmatically.

**Pattern:** Expose a single endpoint (e.g., `GET /man` or `GET /api/docs`) that returns the complete API reference as structured JSON. The response includes an overview of what the API does, the base URL, a catalog of all available endpoints with methods, paths, descriptions, and authentication requirements, complete parameter lists with types and validation rules, example requests with proper formatting, expected responses for success and error cases, and the consistent error format used throughout the API.

When the agent is authenticated, the response can be personalized. Include the agent's current context (which resources they've created, their configuration status), progressive recommendations about what to do next with ready-to-execute examples, and generated examples that interpolate the agent's real credentials instead of placeholders like `YOUR_API_KEY`.

Support topic filtering with query parameters to reduce payload size. For example, `GET /man?topic=resources` returns only endpoints related to resource management. Always include discovery endpoints (the manual itself, health checks, etc.) regardless of filtering.

Support a lightweight guide mode (e.g., `GET /man?guide`) that returns only the overview, current context, and next recommendation without the full endpoint catalog. This dramatically reduces token usage when agents only need orientation.

When unauthenticated, the endpoint works with placeholder values in examples. This allows new agents to discover how to authenticate before they have credentials.

**Trade-offs:**
- Large payload if not filtered. The full API reference can be thousands of tokens. Topic filtering and guide mode mitigate this.
- Couples discovery and documentation. If they drift out of sync, agents receive incorrect information. Generate the manual from code or use strict validation to keep them aligned.
- Must be kept current. Every API change requires updating the manual. Automate this where possible.
- Personalized context and recommendations require querying the agent's state, adding latency and database load.

> See [CalDave implementation](CALDAVE.md#1-self-describing-api-manual) for a production example.

---

### 2. Zero-Auth Agent Bootstrapping

**Tier:** Essential

**Problem:** Traditional APIs require humans to create accounts, navigate dashboards, generate API keys, and configure authentication. AI agents can't perform these steps. Manual provisioning creates a dependency on humans for every new agent.

**Pattern:** Provide a single unauthenticated POST endpoint (e.g., `POST /agents` or `POST /api/keys`) that creates an agent identity and returns a working API key. The endpoint accepts optional metadata like name and description but requires no fields. It generates a unique agent identifier, creates a cryptographically secure API key, stores only a hash of the key (never plaintext), and returns both the agent ID and the API key with a clear warning that the key will only be shown once.

Implement "soft authentication" on discovery endpoints. Soft auth resolves a Bearer token when present but never returns 401 when absent. If the Authorization header is missing or invalid, set the agent context to null and continue processing. This allows the same endpoint to serve both authenticated (personalized) and unauthenticated (generic) responses. Discovery endpoints like the API manual and changelog should use soft auth. Resource endpoints should use hard auth that returns 401 for missing or invalid credentials.

Rate limit the agent creation endpoint aggressively (e.g., 20 requests per hour per IP) to prevent abuse. Consider requiring a CAPTCHA or proof-of-work for additional protection.

Support optional human association at creation time. If a human API key is provided via a custom header (e.g., `X-Human-Key`), automatically link the new agent to that human account for oversight.

**Trade-offs:**
- Abuse potential. Self-registration with no verification is easy to abuse. Rate limiting is essential but not sufficient. Monitor creation patterns and implement additional protections as needed.
- Key shown once means lost keys require re-creating the agent. There's no key recovery mechanism. Document this clearly and consider providing a way to rotate keys for claimed agents.
- Soft auth adds middleware complexity. You need separate logic paths for endpoints that require auth vs allow it vs forbid it.
- Unclaimed agents are orphaned if the creating system loses the key. Consider implementing agent expiration or requiring periodic check-ins.

> See [CalDave implementation](CALDAVE.md#2-zero-auth-agent-bootstrapping) for a production example.

---

### 3. Structured Error Handling

**Tier:** Essential

**Problem:** Agents hallucinate field names, send requests to wrong endpoints, trigger validation errors, and encounter server failures. Inconsistent error formats make programmatic handling difficult. Agents retry the same mistake repeatedly or give up when errors are ambiguous.

**Pattern:** Implement four complementary error handling facets.

**First, use a single consistent error shape** across the entire API. Every error response, regardless of status code, uses the same JSON structure: `{ "error": "Human-readable description of what went wrong" }`. Document this format prominently in your API manual so agents know what to expect before they encounter errors.

**Second, explicitly reject unknown fields**. When agents send write requests (POST, PUT, PATCH), validate the incoming field names against a known set for that resource type. If any field is unrecognized, return a 400 error that lists exactly which fields were unknown: `{ "error": "Unknown fields: start_time, end_time" }`. Never silently ignore fields. Aliases (see pattern 16) should be included in the known set after normalization.

**Third, use the 404 catch-all as a discovery mechanism**. When an agent requests a path that doesn't exist, return a helpful error: `{ "error": "Not found. See GET /man for the API reference." }`. This turns navigation errors into learning opportunities.

**Fourth, implement agent-scoped error introspection**. Log all server errors (500, unexpected exceptions) to a database table with the agent's ID, timestamp, route, method, status code, error message, and stack trace. Expose a `GET /errors` endpoint that returns the authenticated agent's recent errors. Include filtering by route or time range and pagination for long error histories. Only return errors for the authenticated agent to prevent information leakage. This allows agents to debug their own integration issues without human intervention.

**Trade-offs:**
- Unknown field rejection can break forward compatibility. If agents send fields intended for newer API versions, they'll be rejected by older deployments. Use API versioning or feature flags if this is a concern.
- Error introspection requires database storage and careful scoping. Error logs grow over time. Implement retention policies and ensure agents can only access their own errors.
- Specific error messages can leak implementation details. Balance helpfulness with security. Don't expose internal paths, database schemas, or sensitive configuration in error messages.
- Consistent error format limits expressiveness. You can't return structured validation errors with per-field messages without breaking the format. Choose simplicity over flexibility here.

> See [CalDave implementation](CALDAVE.md#3-structured-error-handling) for a production example.

---

### 4. Type-Prefixed IDs

**Tier:** Essential

**Problem:** APIs using UUIDs or numeric IDs make it easy for agents to pass the wrong entity type. When an agent uses a calendar ID where an event ID is expected, the error message says "not found" without explaining the type mismatch. Humans reading logs can't tell what kind of entity an ID represents.

**Pattern:** Prefix every ID with a short string that identifies its entity type. Use a URL-safe alphabet (alphanumeric, no special characters) so IDs can appear in URLs without encoding. Use short IDs (e.g., 12 random characters) for entities that appear in URLs, API responses, and logs. Use long IDs (e.g., 32 random characters) for secrets, tokens, and API keys that need high entropy. Choose a consistent length and alphabet for all IDs of a given category.

Example prefixes: `usr_` for users, `res_` for resources, `itm_` for items, `ctr_` for containers, `evt_` for events, `tok_` for tokens, `key_live_` for API keys, `key_test_` for test mode keys.

Example IDs with prefixes:
```json
{
  "user_id": "usr_x7y8z9AbCdEf",
  "resource_id": "res_a1b2c3XyZwVu",
  "api_key": "key_live_p4q5r6StUvWx...",
  "token": "tok_m7n8o9PqRsTu..."
}
```

When validating ID parameters in endpoints, check the prefix before querying the database. Return specific errors like "Expected resource ID (res_*) but received item ID (itm_*)" when the type doesn't match.

**Trade-offs:**
- Slightly longer IDs due to the prefix. A 4-character prefix on a 12-character random portion yields 16 total characters. This is marginal compared to UUIDs (36 characters with hyphens).
- Prefix is not a security boundary. Don't rely on prefix validation alone. Always verify ownership and permissions even when the prefix is correct.
- Changing prefixes is a breaking change. Choose prefixes carefully. They become part of your API contract.
- Requires consistent discipline. Every ID generation path must use the correct prefix. Centralize ID generation to avoid mistakes.

> See [CalDave implementation](CALDAVE.md#4-type-prefixed-ids) for a production example.

---

### 5. Idempotency Keys

**Tier:** Essential

**Problem:** Agents retry failed requests due to network timeouts, server errors, or conversation context loss. LLM frameworks may re-execute tool calls when regenerating responses. Without idempotency, retries create duplicate resources. Agents create multiple charges, send duplicate notifications, or spawn multiple background jobs from a single logical operation.

**Pattern:** Accept an `Idempotency-Key` header on all mutating endpoints (POST, PUT, PATCH, DELETE). When a request includes this header, the server checks if a request with the same key has been processed recently. If so, return the stored response without re-executing the operation. If not, process the request normally and store the response along with the key for future lookups.

Store idempotency keys with an expiration window (e.g., 24 hours). After expiration, the key can be reused. This prevents unbounded storage growth while providing a reasonable retry window.

When the same key is used with different request parameters (different body, different URL), return a 409 Conflict error explaining that the key has already been used for a different request. This prevents accidental key reuse and catches client bugs.

Scope idempotency keys to the authenticated agent or user. Two different agents can use the same key without collision.

Return a custom header like `Idempotency-Replayed: true` when serving a cached response so clients can distinguish between new operations and replayed results.

Example request:
```bash
curl -X POST https://api.example.com/resources \
  -H "Authorization: Bearer key_live_..." \
  -H "Idempotency-Key: unique-operation-id-12345" \
  -H "Content-Type: application/json" \
  -d '{"name": "My Resource"}'
```

If this request times out and the agent retries with the same `Idempotency-Key`, it receives the original response without creating a second resource.

**Trade-offs:**
- Requires server-side storage for keys and responses. Use a fast key-value store like Redis or a database table with an index on (agent_id, idempotency_key). Implement aggressive expiration to limit storage growth.
- Adds complexity to every write endpoint. Every mutating handler must check for the idempotency key, store the response, and handle replay logic.
- Expiration window is a judgment call. Too short and legitimate retries fail. Too long and storage grows unbounded. 24 hours is a reasonable default.
- Conflict detection (same key, different params) requires storing the full request signature. This adds storage overhead but catches serious bugs.
- Doesn't solve all retry problems. If an agent loses context about which key it used, it can't retry successfully. Keys must be deterministic or persisted client-side.

---

## Important Tier

### 6. Agent-First Ownership Model

**Tier:** Important

**Problem:** Traditional APIs are human-centric. A human creates an account, generates API keys, and grants them to applications. This model requires human involvement for every agent deployment. Agents can't provision themselves or operate autonomously without an owner present.

**Pattern:** Invert the ownership model. Make agents the primary entity. Agents create themselves (see pattern 2), manage their own resources, and operate independently. Human ownership is optional and additive, added after the agent is already functional.

When an agent creates itself via `POST /agents`, it receives an API key and can immediately start using the API. The agent owns its resources. There is no human in the loop.

Provide an optional claiming mechanism. A human who wants oversight or key management can create their own account separately (via a web UI or separate API endpoint). Once authenticated, the human can claim an existing agent by providing the agent's API key: `POST /agents/claim` with the agent's key. This creates an ownership link. The agent continues to function with its existing key, but the human can now view activity, regenerate keys if lost, or revoke access.

Support auto-association at creation time for cases where the human context is already known. If the agent creation request includes a human API key (e.g., via an `X-Human-Key` header), automatically link the new agent to that human. This streamlines the flow for platforms that create agents on behalf of users.

In API responses and admin interfaces, distinguish between claimed and unclaimed agents. Unclaimed agents are fully functional but have no fallback if they lose their key.

**Trade-offs:**
- Unclaimed agents are orphaned if the creating system loses the API key. There's no recovery mechanism. For production systems, consider encouraging or requiring claiming.
- Abuse prevention is harder without human accountability. Unclaimed agents can't be traced back to a responsible party. Rely on rate limiting, monitoring, and terms of service enforcement.
- Claiming flow requires secure verification to prevent unauthorized claims. Use one-time tokens or require possession of the agent's API key to prove ownership.
- Two identity models increase complexity. You need separate tables for agents and humans, a junction table for ownership links, and careful access control to respect both models.

> See [CalDave implementation](CALDAVE.md#6-agent-first-ownership-model) for a production example.

---

### 7. Progressive Enhancement

**Tier:** Important

**Problem:** Forcing agents to configure everything upfront creates a high barrier to entry. Many features may never be needed. Complex setup flows waste tokens and slow down initial integration. Agents abandon APIs that require extensive configuration before basic operations work.

**Pattern:** Design every feature to work with sensible defaults. Required fields should be minimal. Optional fields should be truly optional, not "optional but you'll need them for anything useful."

Start with a minimal creation request that produces a working resource immediately. Then provide incremental configuration endpoints to add capabilities. Each enhancement is independent and can be added in any order.

For container resources, consider including seed data so agents can test retrieval without first creating child resources. Make this opt-out via a flag like `include_sample_data: false`.

Document the enhancement path clearly. When returning a basic resource, include a field like `available_enhancements` that lists optional capabilities with links to documentation.

Example progression:
1. Core functionality: `POST /containers` with just a name. Works immediately for creating child resources.
2. Real-time updates: `PATCH /containers/:id` to add a webhook URL and secret. Now receives change notifications.
3. Custom integrations: `PUT /accounts/notifications` to configure custom notification delivery.
4. External sync: `POST /containers/:id/integrations` to sync with external systems.

Each step is optional. An agent can stop at any point and have a working system.

**Trade-offs:**
- Defaults may not suit all use cases. Choosing good defaults requires understanding common usage patterns. Get this wrong and agents still need to configure everything.
- Progressive complexity means more documentation. You need clear guidance on what each enhancement does and when to use it.
- Seed data can confuse agents expecting empty containers. Make opt-out obvious in the response: `"sample_data_included": true`.
- Testing is more complex with optional features. Every combination of enabled features is a potential code path. Focus on testing common configurations.

> See [CalDave implementation](CALDAVE.md#7-progressive-enhancement) for a production example.

---

### 8. Pre-Computed URLs in Responses

**Tier:** Important

**Problem:** After creating a resource, agents often need derived URLs such as webhook endpoints, feed subscription URLs, email addresses for inbound processing, or links to related resources. Constructing these URLs from components is error-prone. Agents must know the base domain, the correct path structure, how to format tokens, and whether to use query parameters or path segments.

**Pattern:** Include fully assembled, ready-to-use URLs in every response where URLs are relevant. When creating a resource, compute all derived URLs on the server and include them in the response. The agent uses them directly without string manipulation.

For a container resource, include subscription feed URLs with authentication tokens, inbound processing addresses, webhook configuration endpoints, and any other URLs the agent might need. For child resources, include external integration identifiers and links to edit or delete the resource.

Use fully qualified URLs with scheme, domain, path, and any required authentication parameters. Don't return URL fragments or templates that require assembly.

For URLs containing secrets or tokens, include both the full URL (for convenience) and the token separately (for cases where the agent needs to construct URLs manually or rotate tokens).

Example response:
```json
{
  "resource_id": "res_abc123",
  "name": "My Resource",
  "feed_url": "https://api.example.com/feeds/res_abc123?token=tok_xyz789",
  "feed_token": "tok_xyz789",
  "webhook_url": "https://api.example.com/webhooks/res_abc123",
  "edit_url": "https://api.example.com/resources/res_abc123",
  "inbound_email": "res-abc123@inbound.example.com"
}
```

The agent can immediately use these URLs without any knowledge of your domain structure or URL conventions.

**Trade-offs:**
- Responses are larger due to repeated domain names and path structure. This costs tokens. Balance convenience against payload size.
- URLs that include tokens are sensitive. Consider whether to include them directly in responses or require a separate authenticated request to retrieve them.
- If your domain changes, cached responses contain stale URLs. Version your API responses or implement URL redirects to handle domain migrations.
- Full URLs couple the API to a specific domain and protocol. If you need to support multiple domains or protocols, use relative URLs or URL templates instead.

> See [CalDave implementation](CALDAVE.md#8-pre-computed-urls-in-responses) for a production example.

---

### 9. OpenAPI / Structured Tool Schemas

**Tier:** Important

**Problem:** Agent frameworks use function calling and tool use paradigms where the available operations are defined as structured schemas. Without a machine-readable schema, agents rely on free-text descriptions that the LLM must parse and understand. This is error-prone and inconsistent. Developers building agent integrations must manually translate API documentation into tool definitions.

**Pattern:** Publish an OpenAPI 3.x specification for your API. Each endpoint becomes a tool definition with typed parameters, descriptions, constraints, and example values. Use clear operation IDs that become tool names. Include descriptions at the operation, parameter, and schema level.

Serve the OpenAPI spec at a well-known endpoint like `GET /openapi.json`. Many agent frameworks can consume this directly to generate tool definitions. This complements the self-describing API manual (pattern 1). The manual is for runtime discovery and personalization; OpenAPI is for build-time integration and tool generation.

Use OpenAPI's schema features to express validation rules: required vs optional parameters, string formats (email, URL, date-time), numeric ranges, enum values, and array constraints. These become type validation in the agent framework.

Include example values in the schema. Agents can use these as templates when learning how to call your API.

Keep the OpenAPI spec in sync with the actual API. Generate it from code annotations if possible, or use strict validation to catch drift. An incorrect schema is worse than no schema.

**Trade-offs:**
- Keeping the OpenAPI spec in sync with the API requires discipline or tooling. Manual maintenance leads to drift. Code generation or annotation-based approaches are more reliable but add complexity.
- OpenAPI doesn't capture all agent-specific semantics. Recommendations, context, personalization, and progressive guidance aren't part of the spec. Use the self-describing manual for these.
- Two sources of truth if not generated from code. The API implementation and the OpenAPI spec can diverge. Automated testing can catch inconsistencies but requires effort.
- OpenAPI is verbose. The spec file can be large. Serve it compressed and cache aggressively. Consider offering a simplified version with only the most common operations.

---

### 10. Rate Limiting with Standard Headers

**Tier:** Important

**Problem:** Agents need to know rate limits programmatically to implement exponential backoff and avoid 429 errors. Custom or undocumented rate limit headers force agents to reverse-engineer your rate limiting behavior. Without machine-readable limits, agents either poll too aggressively (wasting resources) or too conservatively (introducing latency).

**Pattern:** Use IETF standard rate limit headers (`draft-ietf-httpapi-ratelimit-headers`) on every response. These headers are `RateLimit-Limit` (total requests allowed in the window), `RateLimit-Remaining` (requests remaining in the current window), and `RateLimit-Reset` (seconds until the window resets). Include them on successful responses, not just 429 errors, so agents can preemptively slow down.

Apply different rate limits to different operation types. Creation endpoints should have stricter limits than read endpoints to prevent abuse. Authentication endpoints should have very strict limits to prevent brute force attacks. Inbound webhooks should have generous limits to handle bursts.

Document your rate limiting strategy in the API manual. Include the limits for each operation type, the window duration, whether limits are per-IP or per-agent, and how to handle 429 responses.

When returning a 429 error, include a `Retry-After` header with the number of seconds to wait before retrying. Agents can use this to implement compliant backoff.

Example headers:
```
RateLimit-Limit: 1000
RateLimit-Remaining: 997
RateLimit-Reset: 45
```

An agent can read these on every response and adjust its behavior. When `RateLimit-Remaining` is low, the agent can proactively slow down. When it hits 0, the agent knows to wait `RateLimit-Reset` seconds.

**Trade-offs:**
- Per-IP limits don't work well behind shared proxies or NAT gateways. Multiple agents behind the same IP share a limit. Use authenticated agent IDs for more granular limiting when possible.
- Strict limits on creation endpoints can block legitimate batch provisioning. Consider allowing higher limits for proven agents or implementing request queuing instead of rejection.
- Headers add bytes to every response. Three headers with numeric values add about 80 bytes. This is marginal but not free.
- Standard headers don't cover all scenarios. If you use multiple window types (per-second and per-day) or have endpoint-specific limits, the headers can't express this fully. Document these details in the API manual.

> See [CalDave implementation](CALDAVE.md#10-rate-limiting-with-standard-headers) for a production example.

---

### 11. Pagination

**Tier:** Important

**Problem:** Agents that create resources over time can accumulate thousands of items. Returning all of them in a single response wastes tokens, increases latency, and can exceed LLM context windows. Agents need to retrieve large result sets incrementally.

**Pattern:** Use cursor-based pagination for list endpoints. A cursor is an opaque token that represents a position in the result set. The client includes the cursor in the next request to retrieve the next page.

Return a `next_cursor` field in the response. When there are more results, this field contains the cursor for the next page. When the result set is exhausted, this field is `null`. Also include a `has_more` boolean for clarity.

Accept `cursor` and `limit` query parameters. The `cursor` parameter is the value from the previous response's `next_cursor`. The `limit` parameter controls page size and defaults to a reasonable value like 50 or 100.

Optionally include a `total_count` field if it's cheap to compute. For large datasets, counting all results can be expensive. Make this opt-in via a `include_total=true` parameter.

Prefer cursor-based pagination over offset/limit pagination. Cursors remain stable when items are inserted or deleted during pagination. Offset/limit can skip or duplicate items if the dataset changes between requests.

Implement cursors as opaque tokens. They can be encrypted or signed representations of the sort key and last ID. Don't expose internal IDs or sorting logic in cursors.

Example response:
```json
{
  "items": [
    {"item_id": "itm_abc", "name": "First"},
    {"item_id": "itm_def", "name": "Second"}
  ],
  "next_cursor": "Y3Vyc29yOnZhbHVl",
  "has_more": true,
  "limit": 50
}
```

Next request:
```bash
curl "https://api.example.com/items?cursor=Y3Vyc29yOnZhbHVl&limit=50"
```

**Trade-offs:**
- Cursor-based pagination doesn't support "jump to page N" navigation. Clients must iterate sequentially. This is acceptable for agents but can frustrate human users of your API.
- Opaque cursors frustrate debugging. Developers can't inspect a cursor to understand where they are in the result set. Provide debugging tools or support a verbose mode that explains cursor contents.
- Stateless cursors (encoded sort key and ID) are simpler but leak information about your data model and sorting. Encrypted or signed cursors prevent this but require key management.
- Total count can be expensive to compute on large datasets. If you include it, document the performance implications and consider making it opt-in.

---

### 12. Protocol Abstraction

**Tier:** Important

**Problem:** Interoperating with external systems often requires understanding complex protocols like SMTP, iCal, OAuth flows, or file format parsing. Agents that need to send email must implement MIME multipart messages, handle SMTP authentication, and retry delivery failures. Agents that parse calendar invites must understand iCal format variations across providers. This level of detail is error-prone and wastes agent capability on protocol implementation rather than domain logic.

**Pattern:** Handle all protocol complexity server-side. The agent sends and receives simple JSON. The API translates between JSON and the underlying protocol.

For inbound integration (receiving data from external systems), normalize different provider formats into a consistent JSON shape. When multiple email services send webhooks in different formats, parse each one and return the same structure to the agent. Extract the relevant fields, normalize date formats, handle encoding variations, and present a clean interface.

For outbound integration (sending data to external systems), accept simple JSON and generate the correct protocol format. When an agent needs to send a calendar invite, it posts JSON with title, time, and attendees. The server generates the iCal VCALENDAR format, constructs the MIME multipart message, handles SMTP authentication and delivery, tracks RSVP responses, and reports success or failure back to the agent.

Abstract file formats the same way. If agents need to work with PDFs, images, or spreadsheets, provide endpoints that accept or return JSON representations. The server handles format conversion, validation, and error handling.

**Trade-offs:**
- Abstractions leak. Edge cases in underlying protocols will surface as unexpected behavior. An email provider that doesn't support attachments or an iCal client that misinterprets timezones creates bugs that are hard to diagnose through the abstraction layer.
- Server-side complexity increases significantly. You're now responsible for implementing and maintaining protocol clients, handling retries and failures, normalizing variations across providers, and debugging integration issues.
- Agents lose fine-grained control. If an agent needs to set specific MIME headers or iCal properties that your abstraction doesn't expose, there's no escape hatch. Document the abstraction layer clearly and provide feedback channels for missing capabilities.
- Testing is harder. You need test fixtures for every protocol variation, mock implementations of external services, and integration tests against live APIs. Protocol changes by external providers can break your abstraction.

> See [CalDave implementation](CALDAVE.md#12-protocol-abstraction) for a production example.

---

## Nice-to-Have Tier

### 13. Webhook System

**Tier:** Nice-to-Have

**Problem:** Polling for changes wastes resources and introduces latency. An agent that checks every minute burns 1,440 API calls per day per resource, most of which return "no changes." Time-sensitive operations like event notifications can't rely on polling because the delay is unpredictable.

**Pattern:** Support webhooks so the API can push changes to agents. Implement two categories of webhooks.

**Mutation webhooks** fire immediately when data changes. When a resource is created, updated, or deleted, send a POST request to the agent's configured webhook URL with a payload describing the change. Include the change type (created, updated, deleted), the affected resource ID, the full new state of the resource, and a timestamp.

**Time-based webhooks** fire from a background process when scheduled times arrive. For example, "event starting in 5 minutes" or "subscription expires tomorrow." A poller checks for upcoming triggers and fires webhooks at the appropriate time.

Accept webhook configuration on resource creation or via update endpoints. Required fields are the webhook URL (where to send POST requests) and optionally a webhook secret for HMAC signing. Provide a webhook test endpoint that sends a sample payload so agents can validate their webhook receiver before relying on it.

Sign webhook payloads with HMAC-SHA256 if a secret is configured. Compute the HMAC of the JSON body and include it as a header like `X-Signature`. Document how to verify the signature. Agents should compute the HMAC of the raw body and compare using a timing-safe comparison.

Deliver webhooks asynchronously after sending the API response. Use fire-and-forget delivery: wrap the webhook call in an async function that catches all errors to prevent crashes. Use a reasonable timeout (e.g., 10 seconds) to prevent hanging connections. Log delivery failures but don't retry automatically unless you have a proper queue-based retry system.

**Trade-offs:**
- Webhook receivers must be publicly reachable with valid TLS certificates. Agents behind firewalls or on private networks can't receive webhooks. Provide polling endpoints as a fallback.
- Failed deliveries are lost in fire-and-forget mode. If the agent's webhook receiver is down or slow, it misses the notification. Implementing reliable delivery requires a job queue with retries, dead-letter queues, and monitoring.
- HMAC verification adds implementation burden on receivers. Agents must correctly compute the signature over the raw body without re-serializing JSON. Provide code examples in multiple languages.
- Webhook debugging is hard. When delivery fails, both sides need logs. Provide a webhook delivery log endpoint so agents can see recent attempts, status codes, and error messages.

> See [CalDave implementation](CALDAVE.md#13-webhook-system) for a production example.

---

### 14. Polling Hints

**Tier:** Nice-to-Have

**Problem:** An agent that polls on a fixed interval (e.g., every 5 minutes) wastes API calls when nothing is happening soon and misses time-sensitive data when timing is tight. Without guidance, agents must guess the appropriate polling frequency. Too frequent and they waste resources; too infrequent and they miss events.

**Pattern:** Return machine-readable hints that tell the agent when to poll next. For time-sensitive data, include an ISO 8601 duration indicating when the next interesting event occurs. For low-frequency data, include explicit guidance like "poll weekly."

Use ISO 8601 duration format for precise timing. For example, `PT14M30S` means "14 minutes and 30 seconds." This format is standardized, parseable by libraries in all major languages, and compact (typically 10-20 characters). When an agent queries for upcoming events and the next event starts in 14 minutes and 30 seconds, return `"next_poll_in": "PT14M30S"`.

For endpoints that change infrequently (like changelogs or system status), include human-readable guidance with an approximate frequency: `"poll_recommendation": "Check this endpoint approximately once per week."` This prevents agents from polling hourly for data that only changes monthly.

When there's no upcoming event or change, omit the field or set it to null to indicate that polling is not time-sensitive. The agent can fall back to a default frequency or wait for an external trigger.

**Trade-offs:**
- Hints are advisory, not guaranteed. Agents may ignore them, poll more frequently, or stop polling entirely. Don't rely on polling hints for critical functionality; use webhooks instead.
- Calculated durations add server-side logic and latency. For every response, you're querying for the next event and computing the time difference. Cache these values when possible.
- Stale hints can cause missed events. If an agent receives a response, caches it, and uses the hint an hour later, the hint is wrong. Include the calculation timestamp so agents know when the hint was computed.
- ISO 8601 durations are unfamiliar to some developers. Provide examples and parsing code to ease adoption.

> See [CalDave implementation](CALDAVE.md#14-polling-hints) for a production example.

---

### 15. Batch Operations

**Tier:** Nice-to-Have

**Problem:** Agents often need to create, update, or delete multiple resources in a single logical operation. Making N individual requests means N round trips with network latency for each, N rate limit hits that can cause throttling, N sets of HTTP headers and JSON framing that waste tokens, and N opportunities for partial failures that leave the system in an inconsistent state.

**Pattern:** Accept arrays of operations in a single request. Process them server-side and return per-item results with individual success or failure status. Support batch creation (array of new resources), batch update (array of ID and fields to change), and batch deletion (array of IDs).

For mixed operations (create some, update others, delete others), accept a structured payload with operation type and parameters for each item. Process in the order received unless order doesn't matter, in which case you can parallelize.

Use the same validation and error format as individual endpoints. If item 3 in a batch fails validation, return an error for item 3 with the same structure as a single-item error.

Decide on failure semantics: all-or-nothing (transaction) vs best-effort (partial success). All-or-nothing is safer but can be frustrating if one invalid item blocks 99 valid ones. Best-effort is more forgiving but requires the agent to handle partial failures. Document your choice clearly.

Return a structured response with per-item status:
```json
{
  "results": [
    {
      "index": 0,
      "status": "success",
      "resource": {"resource_id": "res_abc", "name": "First"}
    },
    {
      "index": 1,
      "status": "error",
      "error": "name is required"
    },
    {
      "index": 2,
      "status": "success",
      "resource": {"resource_id": "res_def", "name": "Third"}
    }
  ],
  "summary": {
    "total": 3,
    "succeeded": 2,
    "failed": 1
  }
}
```

The agent can iterate through results and handle each one individually.

**Trade-offs:**
- Batch endpoints are more complex to implement and document. Every validation rule, side effect, and error case must work correctly in batch mode. This roughly doubles the testing surface area.
- Partial failures require careful semantics. If 10 out of 100 items fail, what's the HTTP status code? 200 with errors in the body? 207 Multi-Status? Document the convention clearly.
- Large batches can tie up server resources. Processing 1,000 items in one request can block a worker thread for seconds or minutes. Implement timeouts, limit batch size, or use async processing with a status polling endpoint.
- Rate limiting is ambiguous. Does a batch of 50 items count as 1 request or 50? Document your policy. Consider rate limiting on total items rather than requests.

---

### 16. Field Alias Acceptance

**Tier:** Nice-to-Have

**Problem:** LLMs hallucinate field names based on training data from public codebases. If the term commonly used in StackOverflow, GitHub, or documentation differs from your field name, agents will consistently get it wrong. For example, if your API uses `recurrence` but iCal libraries use `rrule`, agents trained on iCal examples will send `rrule`. If you reject this as an unknown field, the agent is stuck in a correction loop.

**Pattern:** Accept common aliases for fields where confusion is likely. Normalize aliases to canonical field names before validation. Include both the canonical name and all aliases in the known fields set so neither triggers unknown field rejection.

Implement normalization early in the request pipeline. If `rrule` is present and `recurrence` is not, copy `rrule` to `recurrence` and delete `rrule`. Then proceed with validation as usual. The rest of your code only sees canonical field names.

Document both the canonical name and accepted aliases in the API manual. Be explicit about which name is canonical. In responses, always use the canonical name.

Choose aliases based on evidence. If you see agents consistently sending the wrong field name in error logs or support requests, add that as an alias. Don't add aliases speculatively.

**Trade-offs:**
- Aliases create implicit API surface that must be maintained. Once you accept an alias, removing it is a breaking change. Agents come to depend on it.
- Can mask agent misunderstanding. If an agent always uses the alias, it never learns the canonical name. This can cause confusion if the alias is removed or the agent integrates with other systems that use the canonical name.
- Documentation must list both canonical names and aliases. This adds verbosity and can confuse readers about which name to use. Be clear about the recommendation.
- Increases testing surface area. You need to test both canonical names and aliases in all combinations to ensure normalization works correctly.

> See [CalDave implementation](CALDAVE.md#16-field-alias-acceptance) for a production example.

---

### 17. Arbitrary Metadata Field

**Tier:** Nice-to-Have

**Problem:** Different agents have different auxiliary data needs. One agent tracks external CRM IDs. Another stores workflow state. Another attaches tags and priority scores. A rigid schema can't anticipate every agent's needs. Adding fields for every use case bloats the API and creates maintenance burden.

**Pattern:** Add a `metadata` field to resources that accepts arbitrary JSON objects up to a size limit. The API stores the metadata as-is and returns it as-is. No schema validation beyond basic type checking.

Validate that the metadata field is an object (not an array, string, number, boolean, or null). Reject primitive types to enforce structure. Calculate the size of the serialized JSON and enforce a limit (e.g., 16 KB). Beyond that, perform no validation. Store the field in a JSONB column or equivalent and return it exactly as received.

Document the size limit and recommend that agents use the metadata field for small auxiliary data, not large payloads. Suggest alternative solutions (file uploads, external storage) for large data.

If agents need to query by metadata fields, use JSONB query operators (in Postgres) or document-oriented query syntax (in MongoDB). Indexing is possible but performance depends on the database and query patterns.

Example usage:
```bash
curl -X POST https://api.example.com/resources \
  -H "Authorization: Bearer key_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Important Task",
    "metadata": {
      "crm_id": "deal_12345",
      "priority": 0.85,
      "tags": ["urgent", "client"],
      "source": "email_triage",
      "notes": "Follow up by Friday"
    }
  }'
```

The API stores this metadata and returns it in all subsequent responses for this resource.

**Trade-offs:**
- No schema means no validation. Agents can store garbage data. Misspelled keys, wrong types, inconsistent structure. You can't enforce correctness beyond the size limit.
- Querying metadata requires database-specific operators. JSONB queries in Postgres are powerful but not standardized. If you migrate databases, query syntax changes.
- Size limits are a judgment call. Too small and agents can't store what they need. Too large and your database fills with junk. 16 KB accommodates most use cases without enabling abuse.
- Metadata can become a dumping ground. Without structure, agents may store redundant or obsolete data. Consider providing guidance on best practices for metadata usage.

> See [CalDave implementation](CALDAVE.md#17-arbitrary-metadata-field) for a production example.

---

### 18. Agent-Aware Changelog

**Tier:** Nice-to-Have

**Problem:** Agents don't know when new features are added. They continue using the endpoints they were originally programmed with and miss improvements. An agent built six months ago has no idea that webhooks, batch operations, or new query filters are now available. Traditional changelogs are chronological HTML pages designed for humans to skim.

**Pattern:** Expose a structured changelog endpoint (e.g., `GET /changelog`) that returns entries as JSON. When authenticated, split entries into "changes since your agent was created" vs "older changes." This highlights what's new for each agent.

Each entry includes a date, type (feature, improvement, fix, breaking, security), title, description, list of affected endpoints, and links to documentation. Use a type field so agents can programmatically filter. For example, an agent might only care about breaking changes.

When authenticated, fetch the agent's creation timestamp and filter the changelog to entries newer than that date. Return both the full changelog and the filtered "changes since signup" list with a count. Include personalized recommendations based on the agent's current setup. If the agent has calendars but no webhooks configured, recommend setting up webhooks.

Provide explicit polling guidance: `"poll_recommendation": "Check this endpoint approximately once per week."` Changelogs don't update frequently enough to warrant hourly polling.

Use soft auth (pattern 2) so unauthenticated agents can read the changelog without creating an account first.

**Trade-offs:**
- Changelog must be maintained manually unless you can generate it from commit history or release notes. Manual maintenance is error-prone and often forgotten.
- Personalized recommendations require knowledge of agent state. You need to query which features the agent has enabled and suggest relevant new capabilities. This adds complexity and latency.
- Agents must remember to poll the changelog. Without a reminder mechanism, most agents will never check it. Consider including a "changes since last poll" count in other API responses to prompt agents to check.
- Structured format is more verbose than human-readable markdown. Balance machine-parseability with human readability since developers will also read the changelog.

> See [CalDave implementation](CALDAVE.md#18-agent-aware-changelog) for a production example.

---

### 19. Plain Text Rendering

**Tier:** Nice-to-Have

**Problem:** Some agents operate in text-only environments like terminals, chat interfaces, or simple HTTP clients. Parsing JSON to produce a human-readable view adds unnecessary complexity. An agent that needs to show a calendar in a Slack message must parse JSON, format dates, align columns, and construct a text table. This is boilerplate that varies across agents and languages.

**Pattern:** Offer endpoints that return pre-formatted plain text. These are read-only views optimized for display in text-based contexts. Use `text/plain` content type so clients don't try to parse as JSON.

Format output as fixed-width tables with clear column headers, aligned columns, and summary lines. Include relevant metadata like resource names, timestamps, and counts. Make the output readable when piped directly into logs, chat messages, or terminal output.

Provide these endpoints in addition to JSON endpoints, not as replacements. Agents that need programmatic access use JSON. Agents that need display-ready output use plain text.

Example endpoint: `GET /resources/:id/view` returns a formatted table.

Example output:
```
Resources in Container (ctr_abc123)  Owner: usr_xyz
--------------------------------------------------------------------------------
NAME                    CREATED              STATUS      ITEMS
--------------------------------------------------------------------------------
First Resource          2026-04-01 10:00:00  active      12
Second Resource         2026-04-01 14:30:00  active      5
Third Resource          2026-04-02 09:15:00  pending     0
--------------------------------------------------------------------------------
3 resource(s)
```

An agent can fetch this and include it directly in a report or chat message without parsing JSON.

**Trade-offs:**
- Another endpoint to maintain. Every change to the data model requires updating both the JSON endpoint and the plain text formatter.
- Plain text is lossy. You can't round-trip back to structured data. It's read-only and display-only.
- Formatting choices are opinionated. Column widths, date formats, and truncation rules may not suit all consumers. Consider making format customizable via query parameters.
- Fixed-width tables break with long values. If a name is 100 characters, the table becomes unreadable. Implement truncation with ellipsis and document the limits.

> See [CalDave implementation](CALDAVE.md#19-plain-text-rendering) for a production example.

---

### 20. MCP Support

**Tier:** Nice-to-Have

**Problem:** HTTP APIs require agents to construct URLs, set headers, format JSON bodies, parse responses, and handle HTTP errors. This is generic plumbing that every agent must implement. MCP (Model Context Protocol) capable agents prefer structured tool calls where the framework handles HTTP details and the LLM works with high-level operations. Tool use is more natural for LLMs than raw HTTP.

**Pattern:** Expose API capabilities through MCP in addition to the HTTP API. Offer both remote (HTTP-based SSE transport) and local (STDIO transport) modes. Register tools with structured JSON Schema for input validation. Each HTTP endpoint becomes one or more MCP tools with typed parameters and clear descriptions.

Implement a remote MCP endpoint (e.g., `POST /mcp`) that accepts SSE connections and responds to tool calls by invoking the underlying HTTP API. Authentication uses standard Bearer tokens passed via headers. Tools are registered with names like `api_create_resource`, descriptions, and JSON Schema for parameters.

Provide resources for documentation. For example, a `guide://getting-started` resource that returns onboarding content. This content is available to the agent through the MCP client without making HTTP requests.

MCP is additive. The HTTP API remains the primary interface and the source of truth. MCP provides an alternative integration path for agents that support it. The same capabilities, same authentication, same rate limits, same errors. Just a different invocation mechanism.

Include instructions in the MCP server that guide agents through common workflows. For example, "To create a resource, first call api_create_container, then call api_create_resource with the container_id."

**Trade-offs:**
- Two interfaces to maintain. Every new endpoint requires adding an HTTP handler and an MCP tool. Every breaking change affects both. Use code generation or shared definitions to keep them in sync.
- MCP is still evolving. The protocol specification is not yet stable. Expect to update your implementation as the protocol matures.
- Not all agent frameworks support MCP. HTTP remains the universal integration path. MCP is a convenience for supported frameworks.
- Tool granularity decisions affect usability. One tool per HTTP endpoint is simple but verbose. Compound tools (e.g., create_resource_with_items) reduce round trips but increase complexity. Balance based on common use cases.

> See [CalDave implementation](CALDAVE.md#20-mcp-support) for a production example.

---

## About

These patterns were extracted from building [CalDave](https://caldave.ai), a production calendar API for AI agents. They are presented here as generic, reusable patterns applicable to any agent-first API.

CalDave implements all 20 patterns described in this document and has been used in production to validate their effectiveness. The patterns evolved through real-world usage, agent integration challenges, and iterative refinement based on what worked and what didn't.

For detailed examples of each pattern with real code, API calls, and response shapes from a production system, see [CALDAVE.md](CALDAVE.md).

Contributions and feedback welcome via issues and pull requests.
