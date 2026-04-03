# Changelog

## 2026-04-02

### Major restructuring: separated generic patterns from CalDave examples

**What changed:**
- Split README.md into two files: a generic pattern reference (README.md) and a CalDave-specific implementation guide (CALDAVE.md)
- Consolidated 25 patterns into 20 by merging related patterns
- Added 4 new patterns: Idempotency Keys, Pagination, OpenAPI / Structured Tool Schemas, Batch Operations
- Introduced three tiers: Essential (5), Important (7), Nice-to-Have (8)
- Moved Design Principles from end of document to near the top
- Added "If You Implement Nothing Else" section highlighting the 5 essential patterns
- Added trade-offs / "when not to use" section to every pattern
- Replaced all CalDave-specific URLs and code with generic examples using api.example.com

**Pattern consolidations:**

| New Pattern | Absorbed Old Patterns |
|---|---|
| Self-Describing API Manual | Manual, Recommendations, Context Window, Generated Examples |
| Zero-Auth Agent Bootstrapping | Bootstrapping, Soft Auth |
| Structured Error Handling | Unknown Fields, Consistent Format, 404 Discovery, Error Introspection |
| Webhook System | Webhooks, Fire-and-Forget |
| Progressive Enhancement | Progressive Enhancement, Seed Data |

**Why:** The original README conflated two purposes -- a generic pattern reference and a CalDave showcase. The restructure makes the patterns usable as a checklist for anyone building an agent-first API, with CalDave as an optional reference implementation.
