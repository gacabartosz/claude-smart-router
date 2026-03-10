# Smart Router for Claude Code

**FusionRoute-inspired intelligent model routing skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).**

Automatically classifies task complexity and routes to the **cheapest capable model** (Haiku / Sonnet / Opus) via subagents — saving **30-75% on token costs** without sacrificing quality.

![Claude Code Skill](https://img.shields.io/badge/Claude%20Code-Skill-blueviolet)
![License: MIT](https://img.shields.io/badge/License-MIT-green)

## The Problem

Claude Code uses one model for everything. Running Opus to `grep` a file costs **18x more** than using Haiku — for the same result. Over a session, this adds up fast.

## The Solution

Smart Router acts as an intelligent classifier that:

1. **Analyzes** your task's complexity (verb, scope, ambiguity, risk)
2. **Routes** to the cheapest model that can handle it
3. **Decomposes** multi-step tasks and routes each step independently
4. **Escalates** automatically if a cheaper model fails

```
┌─────────────────┐
│   Your Task      │
└────────┬────────┘
         │ classify
    ┌────┴────┐
    ▼         ▼         ▼
┌───────┐ ┌───────┐ ┌───────┐
│ HAIKU │ │SONNET │ │ OPUS  │
│  ⚡    │ │  ⚖️    │ │  🧠   │
│ $0.80 │ │ $3.00 │ │$15.00 │
│ /MTok │ │ /MTok │ │ /MTok │
└───────┘ └───────┘ └───────┘
```

## Routing Logic

| Tier | Model | When | Cost vs Opus |
|------|-------|------|:------------:|
| ⚡ Haiku | Cheapest | Search, read, list, explain, trivial edits | **~5%** |
| ⚖️ Sonnet | Balanced | Code edits, bug fixes, tests, refactoring | **~20%** |
| 🧠 Opus | Full power | Architecture, complex debug, multi-step features | **100%** |

### Classification Signals

| Signal | Low (Haiku) | Medium (Sonnet) | High (Opus) |
|--------|:-----------:|:---------------:|:-----------:|
| **Scope** | 1 file | 2-5 files | 5+ files / unknown |
| **Intent** | read/find/list | fix/add/edit | design/build/migrate |
| **Ambiguity** | Clear, factual | Bounded | Vague, open-ended |
| **Creation** | None | Bounded | Greenfield |
| **Stakes** | None | Moderate | Security/prod/data |

## Installation

### Option 1: Symlink into Claude Code skills

```bash
git clone https://github.com/gacabartosz/claude-smart-router.git ~/.agents/skills/smart-router
ln -s ../../.agents/skills/smart-router ~/.claude/skills/smart-router
```

### Option 2: Copy SKILL.md only

```bash
mkdir -p ~/.claude/skills/smart-router
curl -o ~/.claude/skills/smart-router/SKILL.md \
  https://raw.githubusercontent.com/gacabartosz/claude-smart-router/main/SKILL.md
```

### Option 3: Add to CLAUDE.md

Copy the contents of `SKILL.md` into your project's `CLAUDE.md` file.

## Usage

### Auto-route (recommended)

```
/route find all TODO comments in the project
→ [⚡ HAIKU] Found 23 TODOs across 8 files.

/route fix the pagination bug in the users list
→ [⚖️ SONNET] Fixed: offset was (page * limit) instead of ((page-1) * limit).

/route design a caching strategy for our API
→ [🧠 OPUS] Strategy: Redis response cache + Prisma query cache + event invalidation.
```

### Force a specific tier

```
/route haiku: list all .env files
/route sonnet: add error handling to all controllers
/route opus: debug the memory leak
```

### Multi-step decomposition

```
/route build a user preferences feature with API, UI, and tests

→ Step 1: [⚡ HAIKU]  Explore existing patterns
→ Step 2: [🧠 OPUS]   Design the schema and API
→ Step 3: [⚖️ SONNET] Implement endpoints
→ Step 4: [⚖️ SONNET] Build React UI
→ Step 5: [⚖️ SONNET] Write tests
→ Step 6: [⚡ HAIKU]  Verify everything passes
→ [MIXED] Done. Saved ~40% vs all-Opus.
```

## Estimated Savings

| Your Workload | Always Opus | Smart Routed | Savings |
|--------------|:-----------:|:------------:|:-------:|
| **Explorer** (70% search, 20% edit, 10% complex) | 100% | ~25% | **~75%** |
| **Developer** (40% search, 40% edit, 20% complex) | 100% | ~45% | **~55%** |
| **Architect** (20% search, 30% edit, 50% complex) | 100% | ~65% | **~35%** |

## How It Works Under the Hood

Smart Router is a **prompt-based skill** — no server, no dependencies, no runtime. It's a set of classification rules that Claude Code's main agent follows to decide which `model` parameter to pass to the `Agent` tool.

The key insight from [FusionRoute](https://github.com/xiongny/FusionRoute): you don't need one giant model for everything. A lightweight router that selects the right expert per task outperforms always using the most expensive option.

FusionRoute does this at the **token level** during decoding. Smart Router adapts this for Claude Code at the **task level** — same principle, practical implementation.

### Decision Algorithm (simplified)

```
score = 0
score += intent_score    // read=0, edit=1, design=2
score += scope_score     // 1 file=0, 2-5=1, 5+=2
score += ambiguity_score // clear=0, bounded=1, vague=2
score += creation_score  // none=0, yes=1
score += risk_score      // low=0, medium=1, high=2

if score <= 2: HAIKU
if score <= 5: SONNET
if score >= 6: OPUS
```

## Inspiration

- **[FusionRoute](https://arxiv.org/abs/2601.05106)** — Token-Level LLM Collaboration (Meta AI + CMU, 2026)
- The idea that a lightweight router can outperform always using the most expensive model
- Adapted from token-level expert selection to task-level model routing for practical CLI usage

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- Claude Max plan or API access (for multi-model support)
- Agent tool with `model` parameter support

## License

MIT — use it, fork it, improve it.

---

**Made by [Bartosz Gaca](https://bartoszgaca.pl)** — saving tokens, one route at a time.
