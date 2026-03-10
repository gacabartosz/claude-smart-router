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

> Multi-paper synthesis: [FusionRoute](https://github.com/xiongny/FusionRoute) (Meta AI, 2026),
> [RouteLLM](https://github.com/lm-sys/RouteLLM) (LMSYS, ICLR 2025),
> [TRIM](https://arxiv.org/abs/2601.10245) (step-level routing, 2025),
> [xRouter](https://arxiv.org/html/2510.08439v1) (RL cost-aware, 2025),
> [LLMRouter](https://github.com/ulab-uiuc/LLMRouter) (UIUC, 16+ strategies, 2025)

Task-level model routing for Claude Code: classifies task complexity and delegates
to the cheapest capable model via the `Agent` tool's `model` parameter.

## Core Principle

**Don't use Opus to grep a file. Don't use Haiku to architect a system.**

Every task has a minimum model capability threshold. Routing to that threshold saves
30–75% of token costs without sacrificing quality.

---

## Architecture

```
User Task
    │
    ▼
┌──────────────────────────────┐
│  STAGE 1: FAST RULES (~0ms)  │  ← Pattern matching, zero cost
│  Keyword + scope analysis     │
└──────────┬───────────────────┘
           │ match?
     ┌─────┴─────┐
     │ YES       │ NO
     ▼           ▼
  Route      ┌──────────────────────────────┐
  directly   │  STAGE 2: SCORING (in-agent)  │  ← Multi-dimensional scoring
             │  Intent + Scope + Ambiguity   │
             │  + Creation + Stakes + Context │
             └──────────┬───────────────────┘
                        │
                   ┌────┴────┐
                   ▼         ▼         ▼
               ┌───────┐ ┌───────┐ ┌───────┐
               │ HAIKU │ │SONNET │ │ OPUS  │
               │  ⚡    │ │  ⚖️    │ │  🧠   │
               └───┬───┘ └───┬───┘ └───┬───┘
                   │         │         │
                   └────┬────┘─────────┘
                        ▼
                  ┌───────────────┐
                  │ STAGE 3:      │
                  │ ESCALATION    │  ← If result is inadequate,
                  │ (if needed)   │     auto-retry on next tier
                  └───────────────┘
```

---

## Two-Stage Classification (inspired by claude-router + RouteLLM)

### Stage 1: Fast Rules (zero-cost pattern matching)

Before scoring, check these instant-match patterns:

**→ HAIKU instantly if ALL true:**
- Task verb is: read, find, grep, search, list, show, count, what is, where is, explain, format, convert
- No file creation mentioned
- No words: "all files", "across", "everywhere", "every"
- Single target (one file, one function, one concept)

**→ OPUS instantly if ANY true:**
- Task contains: design, architect, plan, migrate, security audit, vulnerability
- Task is explicitly vague: "make it better", "improve", "optimize" (without specific target)
- Task mentions: "production", "deploy", "breaking change"
- Task scope: "entire codebase", "all services", "whole system"

**→ SONNET instantly if ALL true:**
- Task verb is: fix, add, edit, update, refactor, test, write test, implement
- Specific file(s) or function(s) named
- Single concern

**If no instant match → proceed to Stage 2 scoring.**

### Stage 2: Multi-Dimensional Scoring

Score each dimension 0–2, then sum:

| Dimension | 0 (Low) | 1 (Medium) | 2 (High) |
|-----------|---------|------------|----------|
| **Intent** | read, find, list, explain | fix, add, edit, test, refactor | design, build, migrate, audit |
| **Scope** | 1 file, named target | 2–5 files, named module | 5+ files, unknown, "everywhere" |
| **Ambiguity** | Exact: "line 42", "function X" | Bounded: "the auth module" | Vague: "make it better" |
| **Creation** | No new code | Modify existing code | New files, new module |
| **Stakes** | Dev, local, reversible | Shared code, tests exist | Prod, security, data, irreversible |
| **Context needs** | Self-contained, no history | Some prior conversation | Needs full conversation history |

**Routing thresholds:**
```
score 0–3  → HAIKU
score 4–6  → SONNET
score 7–12 → OPUS
```

**Tie-break rule** (from RouteLLM): when score is exactly on a boundary (3 or 7),
prefer one tier UP. Cost of failure > cost of upgrading.

---

## Step-Level Routing (from TRIM)

For multi-step tasks, don't route the entire task — route EACH STEP independently.

**The TRIM insight**: only critical steps (those likely to derail the solution) need
expensive models. Routine continuation steps can use cheaper models.

### Decomposition Pattern

```
User: "Build a user preferences feature with API, UI, and tests"

Step 1: EXPLORE (what exists?)
  → [⚡ HAIKU] Read existing user model, API patterns, test structure
  → Output: context summary for next steps

Step 2: DESIGN (how should it work?)
  → [🧠 OPUS] Design schema, API contract, UI structure
  → Output: concrete plan with file paths and interfaces

Step 3: IMPLEMENT API (follow the plan)
  → [⚖️ SONNET] Implement endpoints per Opus's plan
  → Output: working API code

Step 4: IMPLEMENT UI (follow the plan)
  → [⚖️ SONNET] Build React component per Opus's plan
  → Output: working UI code

Step 5: WRITE TESTS (pattern-follow)
  → [⚖️ SONNET] Write tests following existing test patterns
  → Output: test files

Step 6: VERIFY (mechanical check)
  → [⚡ HAIKU] Run tests, check lint, verify build
  → Output: pass/fail status
```

### Critical Step Detection (from TRIM)

A step is **critical** (route to higher tier) if:
- It involves **irreversible decisions** (schema design, API contract)
- **Errors cascade** downstream (wrong architecture → all implementation wrong)
- It requires **novel reasoning** (no existing pattern to follow)
- It involves **trade-offs** (performance vs. readability, security vs. convenience)

A step is **routine** (safe for lower tier) if:
- It **follows a plan** already made by a higher tier
- It **follows existing patterns** in the codebase
- It's **mechanical** (run tests, check lint, format code)
- The output is **easily verifiable** (tests pass or they don't)

---

## Orchestration Mode (inspired by claude-router + opusplan)

For complex tasks, use **Opus as strategist + Sonnet/Haiku as executors**:

```
┌─────────────────────┐
│  🧠 OPUS (planner)   │ ← Thinks about WHAT to do
│  Designs the approach │
│  Splits into subtasks │
└──────────┬──────────┘
           │ plan
    ┌──────┴──────┐
    ▼             ▼
┌─────────┐  ┌─────────┐
│⚖️ SONNET │  │⚡ HAIKU  │ ← Does the work
│implement │  │search   │
│test      │  │verify   │
└─────────┘  └─────────┘
```

Invoke with: `/route orchestrate: <complex task>`

The Agent tool call for planning:
```
Agent(
  model: "opus",
  subagent_type: "Plan",
  prompt: "Analyze this task and create a step-by-step execution plan.
           For each step, specify:
           1. What to do (concrete, actionable)
           2. Which tier: haiku/sonnet/opus
           3. What context/files are needed
           4. What output to return
           Task: <user's task>"
)
```

Then execute each step with the tier Opus specified.

---

## Confidence-Based Escalation (from TRIM + xRouter)

If a lower-tier model produces an inadequate result, **escalate automatically**.

### When to escalate:
1. **Agent returns "I'm not sure" or "I can't determine"** → escalate
2. **Result is suspiciously short** (< 2 lines for a code task) → escalate
3. **Agent asks for clarification** instead of answering → escalate
4. **Code has obvious issues** (syntax errors, missing imports) → escalate
5. **Task took multiple retries** at the same tier → escalate

### Escalation protocol:
```
Attempt 1: [⚡ HAIKU] → inadequate result
Attempt 2: [⚖️ SONNET] → include: "Previous attempt by a smaller model was insufficient.
            Here was its output: <haiku_output>. Please provide a better solution."
Attempt 3: [🧠 OPUS] → same pattern, include both prior outputs
```

Show the user: `[⚡→⚖️ ESCALATED] Haiku couldn't handle this, routed to Sonnet.`

### Cost guard (from xRouter):
Only escalate if: `cost_of_retry < cost_of_having_used_higher_tier_from_start`

In practice: always escalate (retry cost is ~50% of total, but success rate increases dramatically).
Don't escalate more than once per step.

---

## Multi-Turn Context Awareness

### Follow-up detection:
If the user's message is a follow-up to a previous routed task:
- **Same topic, minor change** → use SAME tier as before
- **"Also..." / "And..."** → keep current tier
- **"Actually, redesign..."** → re-classify (might need higher tier)
- **"Looks good, now test it"** → likely lower tier (mechanical)

### Context passing:
When spawning agents for follow-up work, include:
- Previous step's output/result
- Relevant file paths discovered earlier
- Decisions made in prior steps

---

## Parallel Routing

When subtasks are independent, launch multiple agents simultaneously
in a SINGLE message (Claude Code runs them in parallel):

```
// All in one message — runs concurrently:
Agent(model: "haiku",  subagent_type: "Explore",
      prompt: "Find all API route definitions in /src")
Agent(model: "haiku",  subagent_type: "Explore",
      prompt: "Read the database schema in /prisma/schema.prisma")
Agent(model: "sonnet", subagent_type: "general-purpose",
      prompt: "Review /src/middleware/auth.ts for security issues")
```

### Parallelization rules:
- **Independent reads** → parallelize on Haiku
- **Independent edits to different files** → parallelize on Sonnet
- **Dependent steps** → sequential (wait for prior result)
- **Max parallel agents**: 4–5 (more causes context confusion)

---

## Subagent Type Mapping

Match the task to the best specialist subagent:

| Task Category | Subagent Type | Typical Tier |
|--------------|--------------|:------------:|
| File search, codebase exploration | `Explore` | ⚡ Haiku |
| General research, Q&A | `general-purpose` | ⚡ Haiku |
| Frontend code (React, CSS, UI) | `Frontend Developer` | ⚖️ Sonnet |
| Backend code (API, DB, server) | `Backend Architect` | ⚖️ Sonnet |
| Full-stack implementation | `Senior Developer` | ⚖️ Sonnet |
| DevOps, CI/CD, infra | `DevOps Automator` | ⚖️ Sonnet |
| Architecture, system design | `Plan` | 🧠 Opus |
| Security audit | `general-purpose` | 🧠 Opus |
| Code review (quality) | `general-purpose` | ⚖️ Sonnet |

---

## Cost Badges

After every routed task, display:

| Badge | Meaning |
|-------|---------|
| `[⚡ HAIKU]` | Cheapest route |
| `[⚖️ SONNET]` | Balanced route |
| `[🧠 OPUS]` | Full power |
| `[⚡→⚖️ ESCALATED]` | Auto-escalated from Haiku to Sonnet |
| `[⚡→🧠 ESCALATED]` | Auto-escalated to Opus |
| `[🧠→⚖️→⚡ ORCHESTRATED]` | Opus planned, others executed |
| `[MIXED ⚡×3 ⚖️×2 🧠×1]` | Decomposed across tiers |

---

## Override Commands

| Command | Effect |
|---------|--------|
| `/route <task>` | Auto-classify and route |
| `/route haiku: <task>` | Force Haiku |
| `/route sonnet: <task>` | Force Sonnet |
| `/route opus: <task>` | Force Opus |
| `/route orchestrate: <task>` | Opus plans, others execute |
| "full power" / "use opus" | Override to Opus |
| "cheap" / "fast" / "save tokens" | Override to Haiku |

---

## Anti-Patterns

1. **Don't route one-liner questions** — just answer directly, agent overhead isn't worth it
2. **Don't route tasks needing full conversation history** — subagents start fresh
3. **Don't downgrade security-critical tasks** — always Opus for security
4. **Don't chain 5+ sequential agents** — overhead exceeds savings; use one Sonnet
5. **Don't forget full context in prompts** — subagents have NO prior conversation
6. **Don't parallelize dependent steps** — wait for outputs before starting next

---

## Context Handoff Checklist

When spawning any subagent, the prompt MUST include:

- [ ] **Working directory** or absolute file paths
- [ ] **Clear instruction** — what to do, specifically
- [ ] **Expected output** — what to return (code, file paths, summary, pass/fail)
- [ ] **Constraints** — what NOT to touch, style requirements
- [ ] **Prior context** — if this is a follow-up, include relevant prior outputs
- [ ] **File contents** — if the agent needs to see code, include it or tell it which files to read

**Bad**: `"fix the bug"` → Agent has no context.
**Good**: `"In /src/auth/login.ts, the validateToken function at line 42 throws TypeError
when token is expired instead of returning null. Fix it to return null for expired tokens.
Do not modify any other functions in the file."`

---

## Pricing Reference (Claude, March 2026)

| Model | Input $/MTok | Output $/MTok | Relative Cost |
|-------|:-----------:|:-------------:|:-------------:|
| Haiku 4.5 | $0.80 | $4.00 | **1×** |
| Sonnet 4.6 | $3.00 | $15.00 | **3.75×** |
| Opus 4.6 | $15.00 | $75.00 | **18.75×** |

### Savings Estimates

| Workload Profile | Always Opus | Smart Routed | Savings |
|-----------------|:-----------:|:------------:|:-------:|
| **Explorer** (70% search, 20% edit, 10% complex) | 100% | ~25% | **~75%** |
| **Developer** (40% search, 40% edit, 20% complex) | 100% | ~45% | **~55%** |
| **Architect** (20% search, 30% edit, 50% complex) | 100% | ~65% | **~35%** |

---

## Examples

### Example 1: Instant Haiku match
```
/route find all TODO comments

Stage 1 match: verb=find, single target, no creation → HAIKU
→ Agent(model: "haiku", subagent_type: "Explore", ...)
→ [⚡ HAIKU] Found 23 TODOs across 8 files.
```

### Example 2: Scored to Sonnet
```
/route fix pagination — page 2 shows same results as page 1

Stage 1: verb=fix, specific bug → SONNET instant match
→ Agent(model: "sonnet", subagent_type: "Senior Developer", ...)
→ [⚖️ SONNET] Fixed offset calculation in src/api/list.ts:28
```

### Example 3: Orchestration
```
/route orchestrate: add user preferences with API, UI, tests

Step 1: [🧠 OPUS Plan]    → Design schema, API, and component structure
Step 2: [⚡ HAIKU Explore] → Read existing patterns (parallel)
Step 3: [⚖️ SONNET impl]   → Implement API endpoints
Step 4: [⚖️ SONNET impl]   → Build preferences React panel
Step 5: [⚖️ SONNET test]   → Write integration tests
Step 6: [⚡ HAIKU verify]  → Run tests & lint
→ [🧠→⚖️→⚡ ORCHESTRATED] Feature complete. ~45% saved vs all-Opus.
```

### Example 4: Escalation
```
/route explain the caching invalidation strategy in this codebase

Try: [⚡ HAIKU] → "I found cache.ts but I'm not sure how invalidation works across services"
Escalate: [⚖️ SONNET] → "The caching uses event-driven invalidation via Redis pub/sub.
  When a record changes in UserService, it publishes to 'user:changed' channel.
  CacheService subscribes and invalidates matching keys. See src/cache/subscriber.ts:15."
→ [⚡→⚖️ ESCALATED] Done.
```

---

## Research References

| Paper / Tool | Key Insight Applied |
|-------------|-------------------|
| [FusionRoute](https://arxiv.org/abs/2601.05106) (Meta, 2026) | Expert selection + complementary correction at each step |
| [RouteLLM](https://arxiv.org/abs/2406.18665) (LMSYS, ICLR 2025) | Threshold-based routing with win rate prediction, 2× savings |
| [TRIM](https://arxiv.org/abs/2601.10245) (2025) | Step-level routing: only critical steps need expensive models, 5× efficiency |
| [xRouter](https://arxiv.org/html/2510.08439v1) (2025) | RL cost-aware routing, Pareto frontier, 80% cost reduction |
| [LLMRouter](https://github.com/ulab-uiuc/LLMRouter) (UIUC, 2025) | 16+ routing strategies (KNN, SVM, Graph, Agentic) |
| [RouterBench](https://arxiv.org/abs/2403.12031) (2024) | Benchmark for multi-LLM routing, 405k inference outcomes |
| [claude-router](https://github.com/0xrdan/claude-router) (2026) | Two-tier rules+LLM classification, orchestration mode |
| Claude Code `opusplan` | Native Opus→Sonnet plan/execute split |
