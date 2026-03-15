# sdd-product — Product Discovery for Agent Teams Lite

A pre-proposal sub-agent that closes the gap between what you have in your head and what the AI actually builds.

Built as an extension for [Agent Teams Lite](https://github.com/Gentleman-Programming/agent-teams-lite) by [Gentleman Programming](https://github.com/Gentleman-Programming) — the orchestrator + sub-agent framework for structured feature development.

---

## The Problem

Agent Teams Lite solves the hard problems of AI-assisted development: context overload, lack of structure, code before agreement, and fragile memory. Its SDD pipeline works brilliantly when you know *what* you want and the uncertainty is *how* to build it.

But here's what actually happens to me: **I get excited about a feature, fire up `/sdd-new`, and realize three phases later that I never properly defined the product.** I skipped the actors, the edge cases, the scope boundaries, the success criteria — all the product thinking that should happen *before* technical planning. And the AI doesn't stop me. It takes my vague prompt and fills every gap with its own assumptions.

You say: *"I want users to be able to export their data."*

The pipeline doesn't ask *"export what data — just their profile, or also their activity history?"* or *"who can trigger the export — the user themselves, or also an admin on their behalf?"* — it just picks answers and keeps going. You end up reviewing a technically solid proposal built on product decisions you never made.

The root cause is not the pipeline — it's **me not doing my product homework**, and the pipeline having no mechanism to force that conversation. There is no structured moment to stop and say *"before we plan the how, let's nail down the what."*

`sdd-explore` answers *"how does the current system work?"* — it reads code, compares approaches, identifies technical risks. `sdd-propose` answers *"what's the technical plan?"* — it creates a proposal with scope, approach, and rollback strategy. Neither is designed to answer *"what exactly are we building, for whom, and with what success criteria?"*

That question falls entirely on you. And if you're like me — if you tend to jump into building before fully defining what you're building — the pipeline happily accelerates you in a direction you never validated. Use cases you didn't consider, actors you didn't intend, scope creep baked in from step one.

## The Solution

`sdd-product` is a new optional phase that sits between exploration and proposal:

```
explore → product? → propose → specs ∥ design → tasks → apply → verify → archive
```

It's a sub-agent specialized in **product discovery and requirement clarification**. Its job is to transform your incomplete intent into a stable product brief that `sdd-propose` can consume with minimal guesswork.

**Important:** `sdd-product` is designed to run *after* `sdd-explore`. It reads the exploration artifact to understand the current system before asking you anything. This is what separates it from a generic PM questionnaire — it knows your codebase, your models, your existing flows. It can still run standalone via `/sdd-product`, but with less context and therefore less targeted questions.

When it has enough clarity, it produces a decision-grade brief (problem, actors, workflow, scope, rules, edge cases, success criteria) and hands off to `sdd-propose`. When material ambiguity remains, it returns `blocked` with targeted questions, the orchestrator relays them to you, and the phase resumes with your answers — without ever breaking the single-thread conversation model.

### What a clarification round looks like

```
You:  /sdd-new user-data-export

AI:   ✓ Explorer analyzed codebase. User model has profile, orders,
        and activity_log tables. No existing export functionality.
        Sidekiq available for background jobs.

      Launching product discovery...

      Product brief (partial):
      ✓ Problem: users have no way to download their own data
      ✓ Primary actor: authenticated end-user
      ✗ Scope unclear — 2 questions need your input:

      Q1: Should the export include only profile data,
          or also order history and activity logs?
          → Profile-only is a 1-day task. Full export
            needs background jobs and adds ~1 week.

      Q2: Should admins be able to trigger an export on
          behalf of a user (e.g. for GDPR requests)?
          → Adds a second actor, a permission check,
            and an audit trail requirement.

You:  Full export, but no admin trigger for now — users self-serve only.

AI:   ✓ Product brief finalized.
      ✓ Launching proposal sub-agent...
```

Two questions. One round. A proposal that reflects what you actually wanted.

---

## How It Fits Into Agent Teams Lite

`sdd-product` was designed to be a native citizen of the existing ecosystem, not an external bolt-on.

### What it reuses (zero reinvention)

- **Compatible envelope.** Returns `{status, executive_summary, detailed_report, artifacts, next_recommended, risks}` — the same structure used by existing ATL skills. No new fields, no schema changes.
- **Same persistence model.** Uses `engram`, `openspec`, `hybrid`, or `none` with the same mode resolution rules from `_shared/persistence-contract.md`. Artifact naming follows the deterministic `sdd/{change-name}/product` convention.
- **Same skill structure.** YAML frontmatter, numbered steps, inline engram calls (`mem_search`, `mem_save`, `mem_get_observation`), mandatory persist step. Compatible with the conventions established in ATL v3.3.x.
- **Same orchestrator mediation.** The sub-agent never talks to the user directly. It returns structured output; the orchestrator relays and resumes. The single-thread conversation model is preserved.
- **Same skill-registry loading.** Step 1 loads the registry from engram or `.atl/skill-registry.md`, just like every other SDD skill.

### What it adds (the new parts)

- **A `product` artifact.** A new node in the persistence layer (`sdd/{change-name}/product`) that `sdd-propose` reads when present. Stored as `product.md` in openspec mode.
- **An optional node in the DAG.** `product?` sits between `explore` and `proposal`. The `?` matters — it's only activated when the orchestrator detects product ambiguity or the user invokes `/sdd-product` directly. Existing flows without product ambiguity are untouched.
- **Blocked-state relay pattern.** When `status: "blocked"`, the orchestrator parses a `## Blocking Questions` section from `detailed_report` and relays questions to the user. This pattern is generic — it works for any future skill that might also need user input mid-phase.
- **A routing heuristic.** Clear criteria for when the orchestrator should activate product (outcome-based requests, unclear actors, undefined scope) and when to skip it (bug fixes, refactors, requests with clear scope).

### What it does NOT change

- No modifications to the envelope schema.
- No modifications to `_shared/persistence-contract.md`, `_shared/engram-convention.md`, or `_shared/openspec-convention.md`.
- No modifications to any existing skill file.
- No new dependencies or tooling.

---

## Compatibility With Existing Skills

| Skill | Overlap? | Notes |
|---|---|---|
| `sdd-explore` | ⚠️ Boundary defined | Explore reads the codebase and reports what exists. Product builds on explore's output to define what we *want*. Product reads explore's artifact to avoid re-reading the same files. If explore didn't run, product does its own limited investigation. |
| `sdd-propose` | ✅ None | Propose benefits from product: instead of inventing requirements, it receives a validated brief. The only change is that propose reads the `product` artifact when present. |
| `sdd-spec` | ✅ None | Spec works from the proposal. Better proposal → better specs. No direct interaction with product. |
| `sdd-design` | ✅ None | Same as spec — indirect benefit, no contact. |
| `sdd-tasks` | ✅ None | Downstream, no interaction. |
| `sdd-apply` | ✅ None | Downstream, no interaction. |
| `sdd-verify` | ✅ None | Downstream, no interaction. |
| `sdd-archive` | ✅ None | Downstream, no interaction. |
| `sdd-init` | ✅ None | Orthogonal — detects stack and bootstraps persistence. |
| `skill-registry` | ✅ None | Product consumes it in Step 1 like every other skill. |

The only skill with a shared surface is `sdd-explore`, and the boundary is clear: explore owns the **current state of the system** (code, patterns, risks); product owns the **desired future state** from a user's perspective (actors, flows, scope, success criteria). Product explicitly avoids re-reading files that explore already analyzed.

---

## Installation

`sdd-product` is a pure Markdown skill. It works with any AI coding agent that supports the Agent Skills standard.

### 1. Copy the skill

```bash
# Claude Code (global)
cp -r skills/sdd-product ~/.claude/skills/sdd-product

# OpenCode (global)
cp -r skills/sdd-product ~/.config/opencode/skills/sdd-product

# Cursor (global or per-project)
cp -r skills/sdd-product ~/.cursor/skills/sdd-product
# or
cp -r skills/sdd-product ./your-project/skills/sdd-product

# Antigravity (global or workspace)
cp -r skills/sdd-product ~/.gemini/antigravity/skills/sdd-product
# or
cp -r skills/sdd-product ./.agent/skills/sdd-product

# Codex
cp -r skills/sdd-product ~/.codex/skills/sdd-product
# or
cp -r skills/sdd-product ./.agents/skills/sdd-product

# Any other tool
# Copy skills/sdd-product/ to wherever your agent reads skill files from.
```

### 2. Update your orchestrator instructions

Apply the deltas described in `ORCHESTRATOR-DELTAS.md` to your orchestrator config — the specific file depends on your agent:

| Agent | Orchestrator file |
|---|---|
| Claude Code | `~/.claude/CLAUDE.md` or project-level `CLAUDE.md` |
| OpenCode | `~/.config/opencode/opencode.json` (agent prompt) |
| Cursor | `.cursorrules` in your project root |
| Antigravity | `~/.gemini/GEMINI.md` or `.agent/rules/sdd-orchestrator.md` |
| Codex | `~/.codex/instructions.md` or agents config |
| Other tools | Wherever your agent reads system prompt or rules |

The deltas are agent-agnostic Markdown — the content is the same regardless of where you paste it:

- Add `/sdd-product <change>` to the commands list.
- Update `/sdd-new` to optionally route through product.
- Add `product?` to the dependency graph.
- Add `sdd-product` to the phase read/write table.
- Add `sdd/{change-name}/product` to the engram topic key table.
- Add the Blocked Phase Handling and Product Ambiguity Detection sections.

### 3. Verify

Open your agent in any project and run:

```
/sdd-product my-feature-name
```

The orchestrator should launch the sub-agent and return a product brief or blocking questions.

> **Note on sub-agent delegation:** Claude Code and OpenCode support true sub-agent delegation via the Task tool (fresh context per phase). Cursor, Antigravity, and Codex run skills inline — the skill still works, but the orchestrator shares context with the sub-agent instead of spawning a fresh one. This is the same trade-off that applies to all Agent Teams Lite skills.

---

## When to Use It

**Use `sdd-product` when:**
- Your request describes a desired outcome, not a concrete feature shape.
- Multiple user roles or permission models could apply.
- Scope boundaries are undefined — what's MVP vs. future.
- The feature involves multi-step user workflows with open design decisions.
- You keep finding that proposals miss your actual intent.

**Skip it when:**
- The request is a bug fix, refactor, or performance improvement.
- Scope and flow are already clear in your initial prompt.
- Uncertainty is purely technical — use `sdd-explore` instead.

---

## Acknowledgments

This skill exists because of [Gentleman Programming](https://github.com/Gentleman-Programming) and the [Agent Teams Lite](https://github.com/Gentleman-Programming/agent-teams-lite) framework. The orchestrator pattern, the skill conventions, the persistence model, the envelope contract — all of that is their work. `sdd-product` is just a new phase that fills a gap in the pipeline while standing on that foundation.

---

## Author

**Pablo Laya** — [prodelaya](https://github.com/prodelaya)

## License

MIT