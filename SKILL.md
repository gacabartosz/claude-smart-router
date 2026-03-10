---
name: smart-router
description: >
  FusionRoute-inspired intelligent model router for Claude Code.
  Automatically classifies task complexity and routes to the cheapest capable model
  (Haiku/Sonnet/Opus) via subagents, saving tokens and money.
  Use when the user wants to run a task cost-efficiently, or invoke with /route.
  Trigger on: "/route", "route this", "smart route", "model router", "cheap mode",
  "save tokens", "cost efficient", "route task".
---

# Smart Router for Claude Code

> Inspired by [FusionRoute](https://github.com/xiongny/FusionRoute) (Meta AI + CMU, 2026) — a token-level multi-LLM collaboration framework that selects the optimal expert at each decoding step.

Adapted for Claude Code as **task-level routing**: classifies task complexity and delegates to the cheapest capable model via the `Agent` tool's `model` parameter.

## Core Principle

**Don't use Opus to grep a file. Don't use Haiku to architect a system.**

Every task has a minimum model capability threshold. Routing to that threshold saves 30-65% of token costs without sacrificing quality.

---

## Architecture

```
User Task
    │
    ▼
┌─────────────────────┐
│   CLASSIFIER         │  ← Runs in the main agent (no extra cost)
│   Analyzes:          │
│   - verb/intent      │
│   - scope (files)    │
│   - ambiguity level  │
│   - creation needed? │
│   - reasoning depth  │
└────────┬────────────┘
         │
    ┌────┴────┐
    ▼         ▼         ▼
┌───────┐ ┌───────┐ ┌───────┐
│ HAIKU │ │SONNET │ │ OPUS  │
│  ⚡    │ │  ⚖️    │ │  🧠   │
│$0.25  │ │$3.00  │ │$15.00 │
│/1M in │ │/1M in │ │/1M in │
└───────┘ └───────┘ └───────┘
    │         │         │
    └────┬────┘─────────┘
         ▼
   Result + Cost Badge
```

---

## Routing Classification

### TIER 1 — Haiku ⚡ (fastest, cheapest)

**When to use**: Single-step, factual, bounded tasks with no ambiguity, no multi-file coordination, no design decisions.

| Category | Keywords / Patterns | Examples |
|----------|-------------------|----------|
| **File search** | find, locate, where is, grep, search for | "find all .env files", "where is the auth config" |
| **File read** | read, show, what's in, cat, print | "show me package.json", "what's in the Dockerfile" |
| **Simple Q&A** | what does, what is, explain, how does X work | "what does this regex do", "explain line 42" |
| **List/enumerate** | list, count, show all, enumerate | "list all React components", "count API endpoints" |
| **Trivial edits** | fix typo, add import, rename, remove comment | "fix the typo in README", "add lodash import" |
| **Git info** | status, log, diff, branch, what changed | "show recent commits", "what files changed" |
| **Format/convert** | format, prettify, convert to JSON/YAML | "convert this to YAML", "format this JSON" |

**Decision signals**:
- Task can be completed with Read, Grep, Glob, or single Edit
- No reasoning chain longer than 1-2 steps
- Answer exists verbatim in the codebase
- No new code creation beyond trivial changes

### TIER 2 — Sonnet ⚖️ (balanced)

**When to use**: Requires creation or editing with clear scope, single concern, may touch 2-5 files, needs moderate reasoning.

| Category | Keywords / Patterns | Examples |
|----------|-------------------|----------|
| **Bounded code edit** | add validation, implement interface, add field | "add email validation to signup form" |
| **Bug fix (clear)** | fix the null check, handle edge case, fix crash in X | "fix the off-by-one error in pagination" |
| **Code review** | review, check for issues, audit this file | "review the auth middleware for bugs" |
| **Refactoring** | extract, simplify, DRY, dedup, clean up | "extract the email logic into a service" |
| **Test writing** | write tests, add unit test, test coverage | "write unit tests for UserService" |
| **Multi-file search** | how does X flow, trace the request path | "trace how auth tokens are validated" |
| **Documentation** | document, add JSDoc, write comments | "add API documentation to routes" |
| **Config/setup** | set up, configure, add dependency | "set up ESLint with Airbnb config" |

**Decision signals**:
- Scope is explicitly bounded (named files, functions, or modules)
- Single concern — one thing to accomplish
- Pattern exists in codebase to follow
- No architectural trade-offs to weigh
- Output is predictable and verifiable

### TIER 3 — Opus 🧠 (most capable)

**When to use**: Ambiguous, multi-file, requires design decisions, deep reasoning, or high-stakes changes.

| Category | Keywords / Patterns | Examples |
|----------|-------------------|----------|
| **Architecture** | design, architect, plan, structure, model | "design the database schema for multi-tenancy" |
| **Complex debug** | intermittent, race condition, memory leak, flaky | "debug why tests fail randomly on CI" |
| **Multi-step feature** | build, implement full, create module, add system | "build a complete notification system" |
| **Ambiguous scope** | make it better, improve, optimize, refactor all | "improve the performance of the dashboard" |
| **Cross-cutting** | add everywhere, implement across, global change | "add structured logging to all services" |
| **Security** | vulnerabilities, audit security, pen test, OWASP | "audit the API for injection vulnerabilities" |
| **Migration** | migrate, upgrade framework, rewrite, port to | "migrate from Express to Fastify" |
| **Planning** | how should we, what's the best approach, strategy | "plan the monolith-to-microservices migration" |

**Decision signals**:
- Task requires weighing trade-offs
- Multiple valid approaches exist
- Scope is unclear or spans entire codebase
- Failure would be costly (security, data loss, production)
- Requires maintaining context across many files
- Creative problem-solving or novel solution needed

---

## Decision Algorithm

```
function route(task):
  // 1. Check explicit overrides
  if task starts with "haiku:" → return HAIKU
  if task starts with "sonnet:" → return SONNET
  if task starts with "opus:" → return OPUS
  if user said "full power" or "use opus" → return OPUS
  if user said "cheap" or "fast" or "save tokens" → return HAIKU

  // 2. Extract signals
  intent    = extractVerb(task)        // read, find, fix, build, design...
  scope     = estimateFileCount(task)  // 1, 2-5, 5+, unknown
  ambiguity = measureAmbiguity(task)   // low, medium, high
  creation  = requiresNewCode(task)    // boolean
  stakes    = assessRisk(task)         // low, medium, high

  // 3. Score each dimension (0-2)
  score = 0
  score += {read/find/list: 0, fix/add/edit: 1, design/build/migrate: 2}[intent]
  score += {1 file: 0, 2-5 files: 1, 5+/unknown: 2}[scope]
  score += {low: 0, medium: 1, high: 2}[ambiguity]
  score += creation ? 1 : 0
  score += {low: 0, medium: 1, high: 2}[stakes]

  // 4. Route by score
  if score <= 2:  return HAIKU   // simple, bounded, safe
  if score <= 5:  return SONNET  // moderate complexity
  if score >= 6:  return OPUS    // complex, ambiguous, high-stakes

  // 5. Tie-break: prefer one tier UP (fail-safe)
  return SONNET
```

---

## Execution Patterns

### Pattern 1: Simple Route (single agent)

```
Agent(
  description: "3-5 word summary",
  prompt: "<full task with all needed context>",
  model: "haiku",           // or "sonnet" or "opus"
  subagent_type: "Explore"  // for search/read tasks
)
```

**Subagent type mapping**:

| Task Type | Subagent Type |
|-----------|--------------|
| Search/read/explore | `Explore` |
| Frontend code | `Frontend Developer` |
| Backend code | `Backend Architect` |
| DevOps/infra | `DevOps Automator` |
| Architecture/planning | `Plan` |
| General coding | `Senior Developer` |
| Testing | `general-purpose` |
| Code review | `general-purpose` |

### Pattern 2: Decomposed Route (multi-step task)

For complex tasks, decompose into subtasks and route EACH to the cheapest capable tier:

```
Example: "Build a REST API with tests and docs"

Step 1: [⚡ HAIKU]  → Explore existing patterns in the codebase
Step 2: [🧠 OPUS]   → Design the API structure and data model
Step 3: [⚖️ SONNET] → Implement the endpoints (following Opus's plan)
Step 4: [⚖️ SONNET] → Write tests
Step 5: [⚡ HAIKU]  → Verify tests pass and lint is clean
Step 6: [⚡ HAIKU]  → Generate API docs from code
```

### Pattern 3: Parallel Route (independent subtasks)

When subtasks don't depend on each other, launch multiple agents simultaneously:

```
// All in one message — runs in parallel:
Agent(model: "haiku",  prompt: "find all API route files")
Agent(model: "haiku",  prompt: "read the database schema")
Agent(model: "sonnet", prompt: "review auth middleware for security issues")
```

### Pattern 4: Escalation Route (retry on failure)

If a lower-tier model fails or produces inadequate results:

```
1. Try HAIKU → result is wrong/incomplete
2. Auto-escalate to SONNET → retry with same prompt + "Previous attempt was insufficient"
3. If still fails → OPUS as final fallback
```

Always inform the user: `[⚡→⚖️ ESCALATED] Haiku couldn't handle this, routed to Sonnet.`

---

## Cost Badges

After every routed task, display the tier badge:

| Badge | Meaning |
|-------|---------|
| `[⚡ HAIKU]` | Cheapest route — simple task |
| `[⚖️ SONNET]` | Balanced route — moderate task |
| `[🧠 OPUS]` | Full power — complex task |
| `[⚡→⚖️ ESCALATED]` | Started cheap, had to escalate |
| `[⚡+⚖️ MIXED]` | Decomposed across tiers |

---

## Override Commands

Users can force a specific tier:

| Command | Effect |
|---------|--------|
| `/route haiku: <task>` | Force Haiku |
| `/route sonnet: <task>` | Force Sonnet |
| `/route opus: <task>` | Force Opus |
| `/route <task>` | Auto-classify |
| "full power" / "use opus" | Override to Opus for this task |
| "cheap" / "fast" / "save tokens" | Override to Haiku |

---

## Pricing Reference (Claude API, March 2025)

| Model | Input | Output | Relative Cost |
|-------|-------|--------|---------------|
| Haiku 4.5 | $0.80/MTok | $4.00/MTok | 1x (baseline) |
| Sonnet 4.6 | $3.00/MTok | $15.00/MTok | ~3.75x |
| Opus 4.6 | $15.00/MTok | $75.00/MTok | ~18.75x |

### Savings Estimates

| Workload Profile | Always Opus | Smart Routed | Savings |
|-----------------|-------------|--------------|---------|
| Explorer (70% search, 20% edit, 10% complex) | 100% | ~25% | **~75%** |
| Developer (40% search, 40% edit, 20% complex) | 100% | ~45% | **~55%** |
| Architect (20% search, 30% edit, 50% complex) | 100% | ~65% | **~35%** |

---

## Anti-Patterns (what NOT to do)

1. **Don't route interactive/conversational tasks** — only route discrete, actionable work
2. **Don't route tasks that need the main conversation context** — subagents don't inherit full history
3. **Don't downgrade security-critical tasks** — always use Opus for security audits
4. **Don't chain 5+ sequential agents** — overhead exceeds savings; use one Sonnet instead
5. **Don't route one-liner questions** — just answer directly, agent overhead isn't worth it
6. **Don't forget to pass full context** — subagents start fresh, include all file paths and requirements

---

## Context Handoff Checklist

When spawning a subagent, always include in the prompt:

- [ ] **Working directory** or specific file paths
- [ ] **What to do** — clear, actionable instruction
- [ ] **What to return** — expected output format
- [ ] **Constraints** — don't modify X, only touch Y
- [ ] **Relevant context** — error messages, function names, branch info

Bad: `"fix the bug"` → Agent has no idea what bug or where.
Good: `"In /src/auth/login.ts, the validateToken function on line 42 throws when token is expired instead of returning null. Fix it to return null for expired tokens.""`

---

## Examples

### Example 1: Codebase exploration
```
User: /route how many React components do we have?

Classification: search + count → HAIKU
→ Agent(model: "haiku", subagent_type: "Explore",
    prompt: "Count all React component files (*.tsx with export default/named component)
             in the current repository. Return the count and list of file paths.")
→ [⚡ HAIKU] Found 47 React components across src/components/ and src/pages/.
```

### Example 2: Bug fix
```
User: /route fix the pagination bug where page 2 shows same results as page 1

Classification: fix + clear scope + bounded → SONNET
→ Agent(model: "sonnet", subagent_type: "Senior Developer",
    prompt: "Fix pagination bug: page 2 shows same results as page 1.
             Check the API endpoint for list queries, look at offset/limit logic.
             Working directory: /Users/me/project/")
→ [⚖️ SONNET] Fixed: offset was calculated as `page * limit` instead of `(page - 1) * limit`
  in src/api/list.ts:28.
```

### Example 3: Architecture
```
User: /route design a caching strategy for our API

Classification: design + ambiguous + cross-cutting → OPUS
→ Agent(model: "opus", subagent_type: "Plan",
    prompt: "Design a caching strategy for the REST API in /Users/me/project/.
             Consider: response caching, database query caching, cache invalidation,
             and the current tech stack. Return a concrete implementation plan.")
→ [🧠 OPUS] Caching strategy designed: Redis for API responses (TTL 5m),
  query-level caching via Prisma middleware, event-driven invalidation via pub/sub.
```

### Example 4: Decomposed task
```
User: /route add a user preferences feature with API, UI, and tests

Classification: multi-step + creation → DECOMPOSE
→ Step 1: [⚡ HAIKU]  Explore existing user model and API patterns
→ Step 2: [🧠 OPUS]   Design the preferences schema and API contract
→ Step 3: [⚖️ SONNET] Implement the API endpoints
→ Step 4: [⚖️ SONNET] Build the React preferences panel
→ Step 5: [⚖️ SONNET] Write integration tests
→ Step 6: [⚡ HAIKU]  Run tests and verify
→ [⚡+⚖️+🧠 MIXED] Feature complete. Saved ~40% vs. all-Opus.
```
