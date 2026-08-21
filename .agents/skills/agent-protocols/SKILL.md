---
name: agent-protocols
description: >-
  Shared protocols for all agents in the multi-agent pipeline: recursive
  nesting, pre-implementation restatement, agent spawn protocol, parallel
  dispatch format, completion reporting, and sovereign subagent awareness.
  Load this skill instead of inlining these protocols in every agent file.
---

# Agent Protocols

Shared behavioral protocols for all agents in the workflow-team pipeline.

## 1. Recursive Nesting Protocol

When your scope card is too broad for a single context:

1. Further decompose using `parallel-dispatch` skill (§1 Decomposition, §5 Hierarchical Decomposition)
2. Spawn sub-agents using pre-registered named TypeNames when available (see §3 Agent Spawn Protocol), or `TypeName="self"` for recursive self-decomposition within the same role
3. Your scope becomes the ceiling — children cannot operate outside it
4. Track sub-agent progress; merge results when all complete
5. Write `.agentwork/handoff.md` for your parent coordinator

Triggers for nesting:
- Task edits >3 unrelated files
- Scope card contains >2 features
- Context approaching 50% capacity
- Secondary expertise needed (delegate to specialist)

## 2. Pre-Implementation Restatement

Before writing code, restate in your own words:
1. What the `.agentwork/brief.md` / scope card asks you to build
2. What files you will create or modify
3. What assumptions you are making

If any assumption is uncertain, document it in your handoff and proceed with the conservative interpretation.

## 3. Agent Spawn Protocol (Coordinators Only)

When spawning ANY agent type with a role file in `.agents/agents/`:

1. **Use the pre-registered named TypeName** matching the agent's role file basename (e.g., `TypeName="backend-engineer"` for `.agents/agents/backend-engineer.md`, `TypeName="tech-lead"` for `.agents/agents/tech-lead.md`). Named types are pre-configured by the platform with appropriate tool sets — builders have write/execution tools, coordinators have write + subagent tools, read-only agents have research tools.
2. **Fall back to `TypeName="self"`** only when no pre-registered type exists for the role, or when recursively sub-decomposing within the same role (e.g., a tech-lead spawning a narrower tech-lead for a sub-scope).
3. **Use `define_subagent`** when you need a custom agent type not covered by pre-registered types. Specify `enable_write_tools`, `enable_subagent_tools`, and `enable_mcp_tools` flags as needed.
4. **Reference the role file** in the system prompt — never paraphrase:
   ```
   "Your role, domain, skills, boundaries, and protocols are defined in
    file://{workspace}/.agents/agents/{agent-type}.md.
   Read this file FIRST before beginning any work."
   ```
5. The child agent MUST read the role file as its first action
6. Propagate this protocol recursively — if the child is a coordinator, it must follow the same rule when spawning its own children

## 4. Parallel Dispatch Format

Each agent file contains a `## Parallel Dispatch` section with role-specific values. The standard fields are:

| Field | Purpose |
|---|---|
| **Scope Axis** | The dimension used to partition work (feature, concern, domain) |
| **Write Scope** | Glob pattern for exclusive write access |
| **Shared Reads** | Glob patterns for read-only access |
| **Constraint** | Key limitation on parallel instances |
| **Integration** | How parallel results are reconciled (if applicable) |

For read-only agents, `Write Scope` becomes `Read Scope` and scoping is for coverage guarantee, not conflict prevention.

## 5. Completion Reporting Protocol

**Every agent MUST report completion to its parent.** This is non-negotiable.

### For code-writing agents (builders, specialists):
1. Write `.agentwork/handoff.md` with status and file manifest
2. Message your parent (the conversation that dispatched you):
   `".agentwork/handoff.md ready — [scope-card-id] [COMPLETE|BLOCKED]"`

### For read-only agents (scouts, reviewers, red team members):
1. Write your deliverable file (findings, verdict, etc.)
2. Message your parent:
   `".agentwork/[deliverable-file] ready — [1-line summary]"`

### Critical: Reply-To Address
Your parent's conversation ID is the conversation that sent you your initial task.
This is the conversation you received your first message from.
**Always reply to THIS conversation ID** — never to any other ID mentioned in your
task description or context.

## 6. Sovereign Subagent Awareness

**Every agent capable of spawning subagents MUST proactively delegate** to specialized builders when the work warrants it. This protocol ensures maximum parallelism, speed, and quality across the pipeline.

### When to spawn sovereign subagents:
- Your scope card spans **multiple domains** (e.g., backend + frontend + tests)
- Your task contains **independently parallelizable** sub-deliverables
- A sub-task requires **secondary expertise** (e.g., you are a tech-lead but the work is pure frontend — delegate to `@frontend-engineer`)
- The `parallel-dispatch` skill's nesting triggers fire (>3 files, >2 features, >50% context)

### When NOT to spawn (do the work yourself):
- The task is **trivially small** (<3 files, single concern, <50 lines of integration code)
- You ARE the specialist for this domain (e.g., you are `@backend-engineer` and it's a backend task)
- Spawning would create **more overhead than value** (single-file fix, quick config change)

### Sovereignty principle:
Each spawned subagent operates **autonomously** within its scope card boundaries. It reads its role file, loads its skills, makes implementation decisions, runs its own quality checks, and reports completion. The parent does NOT micromanage — it decomposes, dispatches, and validates results.

> **Anti-pattern:** A Tech-Lead implementing all backend + frontend + test code itself instead of dispatching `@backend-engineer`, `@frontend-engineer`, and `@test-automation-engineer` in parallel. This defeats the purpose of the multi-agent hierarchy and serializes work that could run concurrently.
