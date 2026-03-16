# ORCHESTRATOR-DELTAS.md — Orchestrator changes for sdd-product

These changes are **agent-agnostic** and aligned with **ATL v4.0.0** conventions (pre-resolved skill registry, inline persistence instructions, word-budgeted artifacts). Apply them to whichever file holds your orchestrator instructions:

| Agent | File |
|---|---|
| Claude Code | `~/.claude/CLAUDE.md` or project-level `CLAUDE.md` |
| OpenCode | Agent prompt in `~/.config/opencode/opencode.json` |
| Cursor | `.cursorrules` in project root |
| Antigravity | `~/.gemini/GEMINI.md` or `.agent/rules/sdd-orchestrator.md` |
| Codex | `~/.codex/instructions.md` or agents config |
| Other tools | System prompt or rules file |

All changes are additive. Nothing existing is removed or replaced.

---

## Delta 1 — Commands (add line)

**Location:** section `### Commands`, after the `/sdd-explore <topic>` line.

**Add:**

```markdown
- `/sdd-product <change>` → launch `sdd-product` sub-agent for product discovery
```

---

## Delta 2 — /sdd-new routing (modify line)

**Location:** section `### Commands`, the `/sdd-new <change>` line.

**Before:**
```markdown
- `/sdd-new <change>` → run `sdd-explore` then `sdd-propose`
```

**After:**
```markdown
- `/sdd-new <change>` → run `sdd-explore`, optionally `sdd-product` if product ambiguity detected, then `sdd-propose`
```

---

## Delta 3 — Dependency Graph (replace graph)

**Location:** section `### Dependency Graph`.

**Before:**
```
proposal -> specs --> tasks -> apply -> verify -> archive
             ^
             |
           design
```

**After:**
```
product? -> proposal -> specs --> tasks -> apply -> verify -> archive
                         ^
                         |
                       design
```

`product?` is optional. If not executed, `proposal` works exactly as before.

---

## Delta 4 — SDD Phase Read/Write Table (add row + modify row)

**Location:** section `### Sub-Agent Context Protocol`, the SDD Phases table.

**Add** as a new row between `sdd-explore` and `sdd-propose`:

```markdown
| `sdd-product` | Exploration (if exists, recommended) + Project context (optional) | Yes (`product`) |
```

**Modify** the `sdd-propose` row:

**Before:**
```markdown
| `sdd-propose` | Exploration (if exists, optional) | Yes (`proposal`) |
```

**After:**
```markdown
| `sdd-propose` | Exploration (if exists, optional) + Product (if exists, optional) | Yes (`proposal`) |
```

---

## Delta 5 — Engram Topic Key Table (add row)

**Location:** section `### Engram Topic Key Format`, the topic keys table.

**Add** after the `Exploration` row:

```markdown
| Product brief | `sdd/{change-name}/product` |
```

---

## Delta 6 — Blocked Phase Handling (new section)

**Location:** after `### Recovery Rule`, at the end of the SDD block.

**Add:**

```markdown
### Blocked Phase Handling

If a sub-agent returns `status: "blocked"`:

1. Parse the `detailed_report` for the `## Blocking Questions` section.
2. Present those questions to the user in the main thread — do NOT rephrase or answer them yourself.
3. When the user replies, relaunch the **same** sub-agent with:
   - The user's answers.
   - The artifact reference (`sdd/{change-name}/{phase}`) so it can resume from its partial work.
   - The same artifact store mode.
4. Repeat until the sub-agent returns `status: "ok"` or `status: "warning"`.
5. Then proceed to the next phase in the dependency graph.

This applies to `sdd-product` and to any future phase that may return `blocked`.
```

---

## Delta 7 — Product Ambiguity Detection (new section)

**Location:** after `### Blocked Phase Handling`.

**Add:**

```markdown
### Product Ambiguity Detection

Before launching `sdd-propose`, evaluate whether the user's request has material product ambiguity. Run `sdd-product` first when:

- The request describes a **desired outcome** rather than a concrete feature shape.
- **User roles, permissions, or flows** are unclear or multiple interpretations exist.
- **Scope boundaries** are not defined (what's in vs. out of MVP).
- The feature involves **multi-step user workflows** with open design decisions.

Skip `sdd-product` when:

- The request is a bug fix, refactor, or performance improvement.
- A clear scope and flow are already stated by the user.
- The uncertainty is purely technical (use `sdd-explore` instead).
```
