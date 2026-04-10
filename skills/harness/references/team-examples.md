# Agent Team Examples

---

## Example 1: Research Team (Agent Team Mode)

### Team Architecture: Fan-out/Fan-in
### Execution Mode: Agent Team

```
[Leader/Orchestrator]
    ├── TeamCreate(research-team)
    ├── TaskCreate(4 research tasks)
    ├── Team members self-coordinate (SendMessage)
    ├── Collect results (Read)
    └── Generate consolidated report
```

### Agent Configuration

| Member | Agent Type | Role | Output |
|--------|-----------|------|--------|
| official-researcher | general-purpose | Official docs/blogs | research_official.md |
| media-researcher | general-purpose | Media/investment | research_media.md |
| community-researcher | general-purpose | Community/SNS | research_community.md |
| background-researcher | general-purpose | Background/competition/academic | research_background.md |
| (Leader = Orchestrator) | — | Consolidated report | final_report.md |

> Research agents use the `general-purpose` built-in type, but must be defined via `.claude/agents/{name}.md` files. These files specify the role, research scope, and team communication protocol to ensure reusability and collaboration quality.

### Orchestrator Workflow (Agent Team)

```
Phase 1: Preparation
  - Analyze user input (identify topic and research mode)
  - Create _workspace/

Phase 2: Team Formation
  - TeamCreate(team_name: "research-team", members: [
      { name: "official", prompt: "Research official channels..." },
      { name: "media", prompt: "Research media/investment trends..." },
      { name: "community", prompt: "Research community reactions..." },
      { name: "background", prompt: "Research background/competitive landscape..." }
    ])
  - TaskCreate(tasks: [
      { title: "Official channel research", assignee: "official" },
      { title: "Media trend research", assignee: "media" },
      { title: "Community reaction research", assignee: "community" },
      { title: "Background environment research", assignee: "background" }
    ])

Phase 3: Research Execution
  - 4 team members investigate independently
  - Interesting findings are shared between members via SendMessage
    (e.g., media shares investment news with background)
  - Conflicting information triggers direct discussion between members
  - Each member saves their file upon completion + notifies the leader

Phase 4: Integration
  - Leader Reads all 4 deliverables
  - Generates consolidated report
  - Conflicting information is annotated with sources

Phase 5: Cleanup
  - Request team members to terminate
  - Clean up team
  - Preserve _workspace/ (for post-verification and audit trails)
```

### Team Communication Pattern

```
official ──SendMessage──→ background  (share relevant official announcements)
media ────SendMessage──→ background  (share investment/acquisition info)
community ─SendMessage──→ media      (media-related info from community reactions)
all members ──TaskUpdate──→ shared task list  (progress updates)
leader ←───── idle notification ──── completed members   (automatic)
```

---

## Example 2: SF Novel Writing Team (Agent Team Mode)

### Team Architecture: Pipeline + Fan-out
### Execution Mode: Agent Team

```
Phase 1 (parallel — agent team): worldbuilder + character-designer + plot-architect
  → Coordinate consistency with each other via SendMessage
Phase 2 (sequential): prose-stylist (writing)
Phase 3 (parallel — agent team): science-consultant + continuity-manager (review)
  → Share findings with each other via SendMessage
Phase 4 (sequential): prose-stylist (revisions incorporating review feedback)
```

### Agent Configuration

| Member | Agent Type | Role | Skill |
|--------|-----------|------|-------|
| worldbuilder | custom | Worldbuilding | world-setting |
| character-designer | custom | Character design | character-profile |
| plot-architect | custom | Plot structure | outline |
| prose-stylist | custom | Style editing + writing | write-scene, review-chapter |
| science-consultant | custom | Scientific verification | science-check |
| continuity-manager | custom | Consistency verification | consistency-check |

### Full Agent File Example: `worldbuilder.md`

```markdown
---
name: worldbuilder
description: "An expert who builds the worldbuilding for SF novels. Designs physics laws, social structures, technology levels, and history."
---

# Worldbuilder — SF Worldbuilding Design Expert

You are a worldbuilding design expert for SF novels. Based on scientific facts but extending with imagination, you build the physical, social, and technological foundations of the world where the story unfolds.

## Core Role
1. Define the world's physics laws and technology level
2. Design social structures, political systems, and economic systems
3. Establish historical context and current conflict structures
4. Describe environments and atmospheres for each location

## Working Principles
- Internal consistency is the top priority — there must be no contradictions between settings
- Use chain questioning like "What if this technology existed?" to infer the world's ripple effects
- Worldbuilding serves the story — avoid excessive settings that hinder the plot

## Input/Output Protocol
- Input: User's worldbuilding concept, genre requirements
- Output: `_workspace/01_worldbuilder_setting.md`
- Format: Markdown. Organized by sections (physics/society/technology/history/locations)

## Team Communication Protocol
- To character-designer: SendMessage social structures, class systems, occupational groups
- To plot-architect: SendMessage the world's major conflict structures and crisis elements
- From science-consultant: Receive scientific error feedback → revise settings
- Broadcast to all relevant team members when worldbuilding changes

## Error Handling
- If the concept is vague, propose 3 directions and request a selection
- When scientific errors are found, present alternatives alongside

## Collaboration
- Provide social structure information to character-designer
- Provide conflict structure information to plot-architect
- Revise settings incorporating feedback from science-consultant
```

### Team Workflow Details

```
Phase 1: TeamCreate(team_name: "novel-team", members: [worldbuilder, character-designer, plot-architect])
         TaskCreate([worldbuilding, character design, plot structure])
         → Team members self-coordinate and work in parallel
         → worldbuilder SendMessages character-designer when social structure is complete
         → character-designer SendMessages plot-architect when protagonist is defined

Phase 2: Clean up Phase 1 team → invoke prose-stylist as a sub-agent (no team needed since it's solo writing)
         prose-stylist Reads the 3 deliverables from _workspace/ and writes
         → Saves result to _workspace/02_prose_draft.md

Phase 3: Create new team — TeamCreate(team_name: "review-team", members: [science-consultant, continuity-manager])
         (Only one team can be active per session, but a new team can be created since the Phase 1 team was cleaned up)
         → Both reviewers review the draft and share findings with each other
         → science-consultant notifies continuity-manager when physics errors are found
         → Clean up team after review is complete

Phase 4: Invoke prose-stylist as a sub-agent, make final revisions incorporating review results
```

---

## Example 3: Webtoon Production Team (Sub-Agent Mode)

### Team Architecture: Generate-Verify
### Execution Mode: Sub-Agent

> In the generate-verify pattern, since there are only 2 agents and result passing is more important than communication, sub-agent mode is appropriate.

```
Phase 1: Agent(webtoon-artist) → Generate panels
Phase 2: Agent(webtoon-reviewer) → Quality review
Phase 3: Agent(webtoon-artist) → Regenerate problematic panels (up to 2 times)
```

### Agent Configuration

| Agent | subagent_type | Role | Skill |
|-------|--------------|------|-------|
| webtoon-artist | custom | Panel image generation | generate-webtoon |
| webtoon-reviewer | custom | Quality review | review-webtoon, fix-webtoon-panel |

### Full Agent File Example: `webtoon-reviewer.md`

```markdown
---
name: webtoon-reviewer
description: "An expert who reviews the quality of webtoon panels. Evaluates composition, character consistency, text readability, and direction."
---

# Webtoon Reviewer — Webtoon Quality Review Expert

You are an expert who reviews the quality of webtoon panels. You evaluate panels based on visual completeness, storytelling effectiveness, and character consistency.

## Core Role
1. Evaluate the composition and visual completeness of each panel
2. Verify character appearance consistency across panels
3. Evaluate speech bubble text readability and placement
4. Review the directing flow and pacing of the entire episode

## Working Principles
- Judge clearly using 3 levels: PASS/FIX/REDO
- FIX is for cases resolvable with partial corrections, REDO requires full regeneration
- Judge based on objective criteria (consistency, readability, composition), not subjective taste

## Input/Output Protocol
- Input: Panel images in the `_workspace/panels/` directory
- Output: `_workspace/review_report.md`
- Format:
  ```
  ## Panel {N}
  - Verdict: PASS | FIX | REDO
  - Reason: [specific reason]
  - Revision instructions: [specific revision direction for FIX/REDO cases]
  ```

## Error Handling
- If image loading fails, mark that panel as REDO
- Panels still marked REDO after 2 regeneration attempts are PASS-ed with a warning

## Collaboration
- Deliver revision instructions to webtoon-artist (file-based)
- Re-review regenerated panels (up to 2 loop iterations)
```

### Error Handling

```
Retry Policy:
- Panels with REDO verdict → request regeneration from artist (with specific revision instructions)
- Force PASS after maximum 2 loops
- If more than 50% of all panels are REDO, suggest prompt revision to the user
```

---

## Example 4: Code Review Team (Agent Team Mode)

### Team Architecture: Fan-out/Fan-in + Discussion
### Execution Mode: Agent Team

> Code review is a representative case where agent teams shine. Reviewers with different perspectives can share findings and challenge each other, enabling deeper reviews.

```
[Leader] → TeamCreate(review-team)
    ├── security-reviewer: Security vulnerability inspection
    ├── performance-reviewer: Performance impact analysis
    └── test-reviewer: Test coverage verification
    → Reviewers share findings with each other (SendMessage)
    → Leader consolidates results
```

### Team Communication Pattern

```
security ──SendMessage──→ performance  ("This SQL query is injectable, needs performance review too")
performance ──SendMessage──→ test      ("Found N+1 query, please check if related tests exist")
test ────SendMessage──→ security      ("No tests for auth module, what's the priority from a security perspective?")
```

Key point: Reviewers communicate **directly without going through the leader**, enabling rapid detection of cross-domain issues.

---

## Example 5: Supervisor Pattern — Code Migration Team (Agent Team Mode)

### Team Architecture: Supervisor
### Execution Mode: Agent Team

```
[supervisor/leader] → Analyze file list → Assign batches
    ├→ [migrator-1] (batch A)
    ├→ [migrator-2] (batch B)
    └→ [migrator-3] (batch C)
    ← Receive TaskUpdate → Assign additional batches or reassign
```

### Agent Configuration

| Member | Role |
|--------|------|
| (Leader = migration-supervisor) | File analysis, batch distribution, progress management |
| migrator-1~3 | Migrate assigned file batches |

### Supervisor's Dynamic Distribution Logic (Using Agent Team)

```
1. Collect complete target file list
2. Estimate complexity (file size, import count, dependencies)
3. Register file batches as tasks via TaskCreate (including dependencies)
4. Team members self-request (claim) tasks
5. When a member reports completion via TaskUpdate:
   - Success → Automatically request next task
   - Failure → Leader confirms cause via SendMessage → Reassign or assign to another member
6. All tasks complete → Leader runs integration tests
```

Difference from fan-out: Tasks are **dynamically assigned at runtime** rather than being fixed in advance. The shared task list's self-request (claim) feature naturally matches the supervisor pattern.

---

## Deliverable Pattern Summary

### Agent Definition Files
Location: `project/.claude/agents/{agent-name}.md`
Required sections: Core role, working principles, input/output protocol, error handling, collaboration
Additional section for team mode: **Team communication protocol** (message send/receive, task request scope)

### Skill File Structure
Location: `project/.claude/skills/{skill-name}/SKILL.md` (project level)
Or: `~/.claude/skills/{skill-name}/SKILL.md` (global level)

### Integration Skill (Orchestrator)
A higher-level skill that coordinates the entire team. Defines agent configurations and workflows per scenario.
Template: See `references/orchestrator-template.md`.
**Execution mode must be explicitly specified** — agent team (default) or sub-agent.
