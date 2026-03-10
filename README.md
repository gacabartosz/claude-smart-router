# Smart Router for Claude Code

**Research-backed intelligent model routing skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).**

Automatically classifies task complexity and routes to the **cheapest capable model** (Haiku / Sonnet / Opus) via subagents — saving **30–75% on token costs** without sacrificing quality.

Built on insights from 7 research papers and tools:
[FusionRoute](https://arxiv.org/abs/2601.05106) · [RouteLLM](https://arxiv.org/abs/2406.18665) · [TRIM](https://arxiv.org/abs/2601.10245) · [xRouter](https://arxiv.org/html/2510.08439v1) · [LLMRouter](https://github.com/ulab-uiuc/LLMRouter) · [RouterBench](https://arxiv.org/abs/2403.12031) · [claude-router](https://github.com/0xrdan/claude-router)

![Claude Code Skill](https://img.shields.io/badge/Claude%20Code-Skill-blueviolet)
![License: MIT](https://img.shields.io/badge/License-MIT-green)

## The Problem

Claude Code uses one model for everything. Running Opus to `grep` a file costs **18.75× more** than Haiku — for the same result. Over a session, this adds up fast.

## The Solution

Smart Router acts as an intelligent classifier that:

1. **Fast-matches** common patterns (zero cost, ~0ms) — inspired by [claude-router](https://github.com/0xrdan/claude-router)
2. **Scores** task complexity across 6 dimensions when rules don't match — inspired by [RouteLLM](https://arxiv.org/abs/2406.18665)
3. **Routes** to the cheapest model that can handle it
4. **Decomposes** multi-step tasks with per-step routing — inspired by [TRIM](https://arxiv.org/abs/2601.10245)
5. **Orchestrates** complex work: Opus plans, Sonnet/Haiku execute — inspired by [FusionRoute](https://arxiv.org/abs/2601.05106)
6. **Escalates** automatically if a cheaper model fails — inspired by [xRouter](https://arxiv.org/html/2510.08439v1)

```
User Task
    │
    ▼
┌──────────────────┐
│ STAGE 1: RULES   │ ← Zero-cost pattern matching
└────────┬─────────┘
    match?│
   ┌─────┴─────┐
   YES         NO
   │    ┌──────────────────┐
   │    │ STAGE 2: SCORING │ ← 6-dimension classification
   │    └────────┬─────────┘
   │             │
   ▼             ▼
┌──────┐ ┌──────┐ ┌──────┐
│HAIKU │ │SONNET│ │ OPUS │
│  ⚡   │ │  ⚖️   │ │  🧠  │
└──┬───┘ └──┬───┘ └──┬───┘
   └────┬────┘────────┘
        ▼
┌──────────────────┐
│ STAGE 3:         │ ← Auto-retry on higher tier
│ ESCALATION       │    if result is inadequate
└──────────────────┘
```

## Key Features

| Feature | Inspired By | What It Does |
|---------|------------|-------------|
| **Two-stage classification** | [claude-router](https://github.com/0xrdan/claude-router) + [RouteLLM](https://arxiv.org/abs/2406.18665) | Fast rules first, multi-dimensional scoring as fallback |
| **6-dimension scoring** | [RouterBench](https://arxiv.org/abs/2403.12031) | Intent, scope, ambiguity, creation, stakes, context needs |
| **Step-level routing** | [TRIM](https://arxiv.org/abs/2601.10245) | Route each step of a multi-step task independently — 5× efficiency |
| **Orchestration mode** | [FusionRoute](https://arxiv.org/abs/2601.05106) + `opusplan` | Opus designs the plan, Sonnet/Haiku execute |
| **Confidence escalation** | [xRouter](https://arxiv.org/html/2510.08439v1) | Auto-retry on higher tier when lower tier fails |
| **Parallel routing** | Claude Code Agent tool | Launch multiple agents simultaneously for independent subtasks |
| **Cost badges** | Original | Show which tier handled each task |

## Routing Logic

| Tier | Model | When | Cost vs Opus |
|------|-------|------|:------------:|
| ⚡ Haiku | Cheapest | Search, read, list, explain, trivial edits, verify | **~5%** |
| ⚖️ Sonnet | Balanced | Code edits, bug fixes, tests, refactoring, reviews | **~20%** |
| 🧠 Opus | Full power | Architecture, complex debug, multi-step features, security | **100%** |

### Scoring Dimensions

| Dimension | 0 (Low) | 1 (Medium) | 2 (High) |
|-----------|---------|------------|----------|
| **Intent** | read, find, list | fix, add, edit, test | design, build, migrate |
| **Scope** | 1 file | 2–5 files | 5+ files / unknown |
| **Ambiguity** | Exact target | Bounded module | Vague / open-ended |
| **Creation** | No new code | Modify existing | New files / module |
| **Stakes** | Dev, reversible | Shared code | Prod, security, data |
| **Context** | Self-contained | Some history | Full conversation needed |

Score 0–3 → Haiku · Score 4–6 → Sonnet · Score 7–12 → Opus

## Installation

### Option 1: Clone + symlink (recommended)

```bash
git clone https://github.com/gacabartosz/claude-smart-router.git ~/.agents/skills/smart-router
ln -s ../../.agents/skills/smart-router ~/.claude/skills/smart-router
```

### Option 2: Download SKILL.md only

```bash
mkdir -p ~/.claude/skills/smart-router
curl -o ~/.claude/skills/smart-router/SKILL.md \
  https://raw.githubusercontent.com/gacabartosz/claude-smart-router/main/SKILL.md
```

### Option 3: Copy into CLAUDE.md

Paste the contents of `SKILL.md` into your project's `CLAUDE.md` file.

## Usage

### Auto-route (recommended)

```
/route find all TODO comments in the project
→ [⚡ HAIKU] Found 23 TODOs across 8 files.

/route fix the pagination bug in the users list
→ [⚖️ SONNET] Fixed offset calculation in src/api/list.ts:28.

/route design a caching strategy for our API
→ [🧠 OPUS] Redis response cache + Prisma middleware + event invalidation.
```

### Force a tier

```
/route haiku: list all .env files
/route sonnet: add error handling to controllers
/route opus: debug the memory leak
```

### Orchestration mode

```
/route orchestrate: add user preferences with API, UI, and tests

→ Step 1: [🧠 OPUS]   Design schema and API contract
→ Step 2: [⚡ HAIKU]  Explore existing patterns
→ Step 3: [⚖️ SONNET] Implement API endpoints
→ Step 4: [⚖️ SONNET] Build React preferences panel
→ Step 5: [⚖️ SONNET] Write tests
→ Step 6: [⚡ HAIKU]  Run tests and verify
→ [🧠→⚖️→⚡ ORCHESTRATED] Done. ~45% saved vs all-Opus.
```

### Auto-escalation

```
/route explain the caching strategy in this codebase

→ [⚡ HAIKU] "Found cache.ts but unsure about cross-service invalidation"
→ [⚡→⚖️ ESCALATED] "Uses Redis pub/sub: UserService publishes to 'user:changed',
   CacheService subscribes and invalidates. See src/cache/subscriber.ts:15."
```

## Estimated Savings

| Your Workload | Always Opus | Smart Routed | Savings |
|--------------|:-----------:|:------------:|:-------:|
| **Explorer** (70% search, 20% edit, 10% complex) | 100% | ~25% | **~75%** |
| **Developer** (40% search, 40% edit, 20% complex) | 100% | ~45% | **~55%** |
| **Architect** (20% search, 30% edit, 50% complex) | 100% | ~65% | **~35%** |

## How It Works

Smart Router is a **prompt-based skill** — no server, no dependencies, no runtime cost.
It's a set of classification rules that Claude Code's agent follows to decide which
`model` parameter to pass to the `Agent` tool.

The full classification algorithm, step-level routing logic, escalation protocol,
orchestration patterns, and anti-patterns are all documented in [`SKILL.md`](SKILL.md).

## Research Foundation

| Paper / Tool | Key Insight Used |
|-------------|-----------------|
| [FusionRoute](https://arxiv.org/abs/2601.05106) (Meta AI, 2026) | Expert selection + complementary correction per step |
| [RouteLLM](https://arxiv.org/abs/2406.18665) (LMSYS, ICLR 2025) | Threshold-based routing, 2× cost savings |
| [TRIM](https://arxiv.org/abs/2601.10245) (2025) | Step-level routing for multi-step reasoning, 5× efficiency |
| [xRouter](https://arxiv.org/html/2510.08439v1) (2025) | RL cost-aware routing, Pareto-optimal cost-quality tradeoff |
| [LLMRouter](https://github.com/ulab-uiuc/LLMRouter) (UIUC, 2025) | 16+ routing strategies (KNN, SVM, Graph, Agentic) |
| [RouterBench](https://arxiv.org/abs/2403.12031) (2024) | Multi-dimensional routing benchmarks, 405k inferences |
| [claude-router](https://github.com/0xrdan/claude-router) (2026) | Two-tier rules + LLM fallback, orchestration mode |

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- Claude Max plan or API access with multi-model support
- Agent tool with `model` parameter

## License

MIT

---

**Made by [Bartosz Gaca](https://bartoszgaca.pl)** — saving tokens, one route at a time.
