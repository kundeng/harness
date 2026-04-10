# Agent Team Design Patterns

## Execution Modes: Agent Teams vs Sub-agents

Understand the key differences between the two execution modes and choose the appropriate one.

### Agent Teams — Default Mode

The team leader creates a team with `TeamCreate`, and teammates run as independent Claude Code instances. Teammates communicate directly via `SendMessage` and self-coordinate through a shared task list (`TaskCreate`/`TaskUpdate`).

```
[Leader] ←→ [Teammate A] ←→ [Teammate B]
  ↕             ↕               ↕
  └───── Shared Task List ──────┘
```

**Core Tools:**
- `TeamCreate`: Create a team + spawn teammates
- `SendMessage({to: name})`: Message a specific teammate
- `SendMessage({to: "all"})`: Broadcast (high cost, use sparingly)
- `TaskCreate`/`TaskUpdate`: Manage the shared task list

**Characteristics:**
- Teammates can directly converse, challenge, and verify each other
- Information exchange between teammates without going through the leader
- Self-coordination via shared task list (teammates can self-assign tasks)
- Teammates automatically notify the leader when they become idle
- Plan approval mode allows review before risky operations

**Constraints:**
- Only one team can be **active** per session (however, teams can be disbanded and new ones formed between phases)
- No nested teams (a teammate cannot create its own team)
- Leader is fixed (cannot be transferred)
- High token cost

**Team Restructuring Pattern:**
When different phases require different expert combinations, follow this sequence: save the previous team's artifacts to files → disband the team → create a new team. Previous team artifacts are preserved in `_workspace/`, so the new team can access them via Read.

### Sub-agents — Lightweight Mode

The main agent creates sub-agents using the `Agent` tool. Sub-agents return results only to the main agent and do not communicate with each other.

```
[Main] → [Sub A] → Return result
       → [Sub B] → Return result
       → [Sub C] → Return result
```

**Core Tools:**
- `Agent(prompt, subagent_type, run_in_background)`: Create a sub-agent

**Characteristics:**
- Lightweight and fast
- Results are summarized and returned to the main context
- Token-efficient

**Constraints:**
- No communication between sub-agents
- Main agent handles all coordination
- No real-time collaboration or challenge possible

### Mode Selection Decision Tree

```
Are there 2 or more agents?
├── Yes → Is inter-agent communication needed?
│         ├── Yes → Agent Team (default)
│         │         Cross-verification, discovery sharing, and real-time
│         │         feedback improve quality.
│         │
│         └── No → Sub-agents are also viable
│                  Only result passing is needed, e.g., producer-reviewer,
│                  expert pool, etc.
│
└── No (1 agent) → Sub-agent
                   A single agent does not need team setup.
```

> **Core Principle:** Agent teams are the default. When choosing sub-agents, ask yourself: "Is inter-teammate communication truly unnecessary?"

---

## Agent Team Architecture Types

### 1. Pipeline
Sequential workflow. The output of the previous agent becomes the input for the next.

```
[Analysis] → [Design] → [Implementation] → [Verification]
```

**Best for:** Each stage strongly depends on the artifacts of the previous stage
**Example:** Novel writing — World-building → Characters → Plot → Writing → Editing
**Caution:** A bottleneck delays the entire pipeline. Design each stage to be as independent as possible.
**Team mode suitability:** Strong sequential dependencies limit team mode benefits. However, team mode is useful if there are parallel segments within the pipeline.

### 2. Fan-out/Fan-in
Parallel processing followed by result aggregation. Independent tasks are performed simultaneously.

```
         ┌→ [Expert A] ─┐
[Dispatch] → ├→ [Expert B] ─┼→ [Aggregate]
         └→ [Expert C] ─┘
```

**Best for:** When the same input requires analysis from different perspectives or domains
**Example:** Comprehensive research — simultaneous investigation of official sources, media, community, and background → integrated report
**Caution:** The quality of the aggregation stage determines overall quality.
**Team mode suitability:** The most natural pattern for agent teams. **Must be implemented as an agent team.** Teammates share discoveries and challenge each other, and one agent's findings can redirect another agent's investigation in real time, significantly improving quality compared to isolated investigation.

### 3. Expert Pool
Selectively invokes the appropriate expert based on the situation.

```
[Router] → { Expert A | Expert B | Expert C }
```

**Best for:** When different processing is needed depending on input type
**Example:** Code review — invoke only the relevant expert(s) among security, performance, and architecture specialists
**Caution:** The router's classification accuracy is critical.
**Team mode suitability:** Sub-agents are more suitable. Since only the needed experts are invoked, a permanent team is unnecessary.

### 4. Producer-Reviewer
A producer agent and a reviewer agent operate as a pair.

```
[Producer] → [Reviewer] → (if issues) → [Producer] re-run
```

**Best for:** When artifact quality assurance is important and objective verification criteria exist
**Example:** Webtoon — artist produces → reviewer inspects → problematic panels are regenerated
**Caution:** A maximum retry limit (2-3 times) is mandatory to prevent infinite loops.
**Team mode suitability:** Agent teams are useful. Real-time feedback exchange between producer and reviewer via SendMessage.

### 5. Supervisor
A central agent manages task state and dynamically distributes work to subordinate agents.

```
         ┌→ [Worker A]
[Supervisor] ─┼→ [Worker B]    ← Supervisor monitors state and distributes dynamically
         └→ [Worker C]
```

**Best for:** When workload is variable or task distribution must be decided at runtime
**Example:** Large-scale code migration — supervisor analyzes the file list and assigns batches to workers
**Difference from fan-out:** Fan-out distributes tasks in advance with fixed assignments; supervisor adjusts dynamically based on progress
**Caution:** Set delegation units large enough so the supervisor does not become a bottleneck.
**Team mode suitability:** The shared task list of agent teams naturally matches the supervisor pattern. Register tasks with TaskCreate, and teammates self-assign.

### 6. Hierarchical Delegation
Upper-level agents recursively delegate to lower-level agents. Complex problems are decomposed step by step.

```
[Director] → [Team Lead A] → [Worker A1]
                            → [Worker A2]
             → [Team Lead B] → [Worker B1]
```

**Best for:** Problems that naturally decompose into a hierarchical structure
**Example:** Full-stack app development — Director → Frontend Lead → (UI/Logic/Tests) + Backend Lead → (API/DB/Tests)
**Caution:** Beyond 3 levels of depth, latency and context loss increase significantly. 2 levels or fewer is recommended.
**Team mode suitability:** Agent teams cannot be nested (a teammate cannot create a team). Implement level 1 as a team and level 2 as sub-agents, or flatten into a single team.

## Composite Patterns

In practice, composite patterns are more common than single patterns:

| Composite Pattern | Composition | Example |
|----------|------|------|
| **Fan-out + Producer-Reviewer** | Parallel production followed by individual review | Multilingual translation — 4 languages translated in parallel → each reviewed by a native reviewer |
| **Pipeline + Fan-out** | Some stages within a sequential pipeline are parallelized | Analysis (sequential) → Implementation (parallel) → Integration testing (sequential) |
| **Supervisor + Expert Pool** | Supervisor dynamically invokes experts | Customer inquiry handling — supervisor classifies the inquiry then assigns the appropriate expert |

### Execution Mode for Composite Patterns

**Use agent teams for all composite patterns by default.** Active communication between teammates is the key driver of result quality.

| Scenario | Recommended Mode | Reason |
|---------|----------|------|
| **Research + Analysis** | Agent Team | Investigators share discoveries, real-time discussion of conflicting information |
| **Design + Implementation + Verification** | Agent Team | Feedback loops between designer, implementer, and verifier |
| **Supervisor + Workers** | Agent Team | Dynamic assignment via shared task list, workers share progress with each other |
| **Production + Review** | Agent Team | Real-time feedback between producer and reviewer minimizes rework |

> Consider mixing in sub-agents only when a single agent performs a completely isolated, one-off task.

## Agent Type Selection

When invoking an agent, specify the type using the `subagent_type` parameter of the Agent tool. Agent team teammates can also use custom agent definitions.

### Built-in Types

| Type | Tool Access | Best For |
|------|----------|-----------|
| `general-purpose` | Full access (including WebSearch, WebFetch) | Web research, general-purpose tasks |
| `Explore` | Read-only (no Edit/Write) | Codebase exploration, analysis |
| `Plan` | Read-only (no Edit/Write) | Architecture design, planning |

### Custom Types

Define an agent in `.claude/agents/{name}.md` and invoke it with `subagent_type: "{name}"`. Custom agents have full tool access.

### Selection Criteria

| Situation | Recommended | Reason |
|------|------|------|
| Complex role reused across multiple sessions | **Custom type** (`.claude/agents/`) | Manage persona and work principles as a file |
| Simple research/collection where a prompt suffices | **`general-purpose`** + detailed prompt | No agent file needed; instructions included in the prompt |
| Only code reading needed (analysis/review) | **`Explore`** | Prevents accidental file modifications |
| Only design/planning needed | **`Plan`** | Focuses on analysis, prevents code changes |
| Implementation tasks requiring file modifications | **Custom type** | Full tool access + specialized instructions |

**Principle:** All agents must be defined in `.claude/agents/{name}.md` files. Even for built-in types, create an agent definition file to specify roles, principles, and protocols. Files must exist for reuse in future sessions, and team communication protocols must be specified to ensure collaboration quality.

**Model:** All agents use `model: "opus"`. Always specify the `model: "opus"` parameter when calling the Agent tool.

## Agent Definition Structure

```markdown
---
name: agent-name
description: "1-2 sentence role description. List trigger keywords."
---

# Agent Name — One-line Role Summary

You are a [role] expert in [domain].

## Core Roles
1. Role 1
2. Role 2

## Work Principles
- Principle 1
- Principle 2

## Input/Output Protocol
- Input: [Where and what is received]
- Output: [Where and what is written]
- Format: [File format, structure]

## Team Communication Protocol (Agent Team Mode)
- Receiving messages: [From whom and what messages are received]
- Sending messages: [To whom and what messages are sent]
- Task requests: [What types of tasks are requested from the shared task list]

## Error Handling
- [Behavior on failure]
- [Behavior on timeout]

## Collaboration
- Relationships with other agents
```

## Agent Separation Criteria

| Criterion | Separate | Merge |
|------|------|------|
| Expertise | Separate if domains differ | Merge if domains overlap |
| Parallelism | Separate if independent execution is possible | Consider merging if sequentially dependent |
| Context | Separate if context burden is large | Merge if lightweight and fast |
| Reusability | Separate if used by other teams | Consider merging if used only by this team |

## Skills vs Agents

| Aspect | Skill | Agent |
|------|-------------|-----------------|
| Definition | Procedural knowledge + tool bundle | Expert persona + behavioral principles |
| Location | `.claude/skills/` | `.claude/agents/` |
| Trigger | User request keyword matching | Explicit invocation via Agent tool |
| Size | Small to large (workflow) | Small (role definition) |
| Purpose | "How to do it" | "Who does it" |

A skill is a **procedural guide** that agents reference when performing tasks.
An agent is an **expert role definition** that utilizes skills.

## Skill-Agent Connection Methods

Three ways an agent can utilize a skill:

| Method | Implementation | Best For |
|------|------|-----------|
| **Skill tool invocation** | Specify `Invoke /skill-name via Skill tool` in the agent prompt | When the skill is an independent workflow and can be invoked by users |
| **Inline in prompt** | Include skill content directly in the agent definition | When the skill is short (under 50 lines) and exclusive to this agent |
| **Reference loading** | Load skill's references/ files via `Read` as needed | When skill content is large and only conditionally needed |

Recommendation: Use Skill tool for high reusability, inline for exclusive use, reference loading for large content.
