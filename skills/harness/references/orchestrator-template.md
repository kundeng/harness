# Orchestrator Skill Template

An orchestrator is a top-level skill that coordinates the entire team. Three templates are provided by execution mode:

- **Template A: Agent Team Mode (Default)** — The preferred choice when 2 or more agents collaborate
- **Template B: Sub-Agent Mode (Alternative)** — When team communication is unnecessary
- **Template C: Hybrid Mode** — Mix modes per Phase

---

## Template A: Agent Team Mode (Default / Preferred Choice)

The **default mode to consider first** when 2 or more agents collaborate. Use `TeamCreate` to form the team, and coordinate through a shared task list and `SendMessage`.

```markdown
---
name: {domain}-orchestrator
description: "Orchestrator that coordinates the {domain} agent team. {initial execution keywords}. Follow-up tasks: This skill MUST also be used when requesting {domain} result modifications, partial re-execution, updates, supplements, re-runs, or improvements to previous results."
---

# {Domain} Orchestrator

An integrated skill that coordinates the {domain} agent team to produce {final artifact}.

## Execution Mode: Agent Team

## Agent Configuration

| Teammate | Agent Type | Role | Skill | Output |
|----------|-----------|------|-------|--------|
| {teammate-1} | {custom or built-in} | {role} | {skill} | {output-file} |
| {teammate-2} | {custom or built-in} | {role} | {skill} | {output-file} |
| ... | | | | |

## Workflow

### Phase 0: Context Check (Follow-up Task Support)

Check whether existing artifacts exist to determine the execution mode:

1. Check whether `_workspace/` directory exists
2. Determine execution mode:
   - **`_workspace/` does not exist** → Initial execution. Proceed to Phase 1
   - **`_workspace/` exists + user requests partial modification** → Partial re-execution. Re-invoke only the relevant agent, overwriting only the target artifacts among existing ones
   - **`_workspace/` exists + new input provided** → New execution. Move existing `_workspace/` to `_workspace_{YYYYMMDD_HHMMSS}/` then proceed to Phase 1
3. For partial re-execution: Include previous artifact paths in the agent prompt so the agent reads existing results and incorporates feedback

### Phase 1: Preparation
1. Analyze user input — {what is being identified}
2. Create `_workspace/` in the working directory (on initial execution)
3. Save input data to `_workspace/00_input/`

### Phase 2: Team Formation

1. Create team:
   ```
   TeamCreate(
     team_name: "{domain}-team",
     members: [
       { name: "{teammate-1}", agent_type: "{type}", model: "opus", prompt: "{role description and task instructions}" },
       { name: "{teammate-2}", agent_type: "{type}", model: "opus", prompt: "{role description and task instructions}" },
       ...
     ]
   )
   ```

2. Register tasks:
   ```
   TaskCreate(tasks: [
     { title: "{task-1}", description: "{details}", assignee: "{teammate-1}" },
     { title: "{task-2}", description: "{details}", assignee: "{teammate-2}" },
     { title: "{task-3}", description: "{details}", depends_on: ["{task-1}"] },
     ...
   ])
   ```

   > 5-6 tasks per teammate is optimal. Specify `depends_on` for tasks with dependencies.

### Phase 3: {Main Task — e.g., Research/Generation/Analysis}

**Execution method:** Teammates self-coordinate

Teammates claim tasks from the shared task list and perform them independently.
The leader monitors progress and intervenes when necessary.

**Inter-teammate communication rules:**
- {teammate-1} sends {what information} to {teammate-2} via SendMessage
- {teammate-2} saves results to a file upon task completion and notifies the leader
- If a teammate needs another teammate's results, they request via SendMessage

**Artifact storage:**

| Teammate | Output Path |
|----------|------------|
| {teammate-1} | `_workspace/{phase}_{teammate-1}_{artifact}.md` |
| {teammate-2} | `_workspace/{phase}_{teammate-2}_{artifact}.md` |

**Leader monitoring:**
- Receives automatic notifications when a teammate becomes idle
- Sends instructions via SendMessage or reassigns tasks when a specific teammate is stuck
- Checks overall progress via TaskGet

### Phase 4: {Follow-up Task — e.g., Verification/Integration}
1. Wait for all teammates to complete their tasks (check status via TaskGet)
2. Collect each teammate's artifacts via Read
3. {Integration/verification logic}
4. Generate final artifact: `{output-path}/{filename}`

### Phase 5: Cleanup
1. Request teammates to terminate (SendMessage)
2. Clean up team (TeamDelete)
3. Preserve `_workspace/` directory (do not delete intermediate artifacts — for post-review and audit trail)
4. Report result summary to user

> **When team restructuring is needed:** If different expert combinations are required per Phase, clean up the current team with TeamDelete, then form the next Phase's team with a new TeamCreate. Previous team's artifacts are preserved in `_workspace/`, so the new team can access them via Read.

## Data Flow

```
[Leader] → TeamCreate → [teammate-1] ←SendMessage→ [teammate-2]
                          │                           │
                          ↓                           ↓
                    artifact-1.md              artifact-2.md
                          │                           │
                          └───────── Read ────────────┘
                                     ↓
                              [Leader: Integration]
                                     ↓
                              Final Artifact
```

## Error Handling

| Situation | Strategy |
|-----------|----------|
| 1 teammate fails/stops | Leader detects → checks status via SendMessage → restart or create replacement teammate |
| Majority of teammates fail | Notify user and confirm whether to proceed |
| Timeout | Use partial results collected so far, terminate incomplete teammates |
| Data conflict between teammates | Annotate sources and present both, do not delete |
| Task status delay | Leader checks via TaskGet then manually uses TaskUpdate |

## Test Scenarios

### Normal Flow
1. User provides {input}
2. Phase 1 produces {analysis results}
3. Phase 2 forms team ({N} teammates + {M} tasks)
4. Phase 3: teammates self-coordinate and perform tasks
5. Phase 4 integrates artifacts to produce final result
6. Phase 5 cleans up team
7. Expected result: `{output-path}/{filename}` is created

### Error Flow
1. {teammate-2} stops due to error in Phase 3
2. Leader receives idle notification
3. Checks status via SendMessage → attempts restart
4. If restart fails, reassigns {teammate-2}'s tasks to {teammate-1}
5. Proceeds to Phase 4 with remaining results
6. Final report states "{teammate-2} area partially uncollected"
```

---

## Template B: Sub-Agent Mode (Alternative)

For cases where team communication overhead is unnecessary. Call directly with the `Agent` tool and collect results via return values.

```markdown
---
name: {domain}-orchestrator
description: "Orchestrator that coordinates {domain} agents. {initial execution keywords}. Include follow-up task keywords."
---

## Execution Mode: Sub-Agent

## Agent Configuration

| Agent | subagent_type | Role | Skill | Output |
|-------|--------------|------|-------|--------|
| {agent-1} | {built-in or custom} | {role} | {skill} | {output-file} |
| {agent-2} | ... | ... | ... | ... |

## Workflow

### Phase 0: Context Check
(Same as Template A — branch based on `_workspace/` existence)

### Phase 1: Preparation
1. Analyze input
2. Create `_workspace/` (on initial execution)

### Phase 2: Parallel Execution
Invoke N Agent tools simultaneously in a single message:

| Agent | Input | Output | model | run_in_background |
|-------|-------|--------|-------|-------------------|
| {agent-1} | {source} | `_workspace/{phase}_{agent}_{artifact}.md` | opus | true |
| {agent-2} | {source} | `_workspace/{phase}_{agent}_{artifact}.md` | opus | true |

### Phase 3: Integration
1. Collect return values from each agent
2. Collect file-based artifacts via Read
3. Apply integration logic → final artifact

### Phase 4: Cleanup
1. Preserve `_workspace/`
2. Report result summary

## Error Handling
- 1 agent fails: Retry once. If it fails again, note the omission and proceed
- Majority fail: Notify user and confirm whether to proceed
- Timeout: Use partial results collected so far
```

---

## Template C: Hybrid Mode

Use different execution modes per Phase. Specify `**Execution Mode:** {Team | Sub}` at the top of each Phase.

```markdown
---
name: {domain}-orchestrator
description: "{domain} orchestrator (hybrid). {keywords}. Include follow-up task keywords."
---

## Execution Mode: Hybrid

| Phase | Mode | Reason |
|-------|------|--------|
| Phase 2 (Parallel Collection) | Sub-Agent | Independent data collection, no team communication needed |
| Phase 3 (Consensus Integration) | Agent Team | Conflicting data requires discussion and consensus |
| Phase 4 (Independent Verification) | Sub-Agent | Single QA agent performs objective verification |

## Workflow

### Phase 2: Parallel Data Collection
**Execution Mode:** Sub-Agent

Invoke N agents in parallel via Agent tool in a single message (`run_in_background: true`).
Each result is saved to `_workspace/02_{agent}_raw.md`.

### Phase 3: Consensus-Based Integration
**Execution Mode:** Agent Team

1. Form integration team with `TeamCreate` (editor + fact-checker + synthesizer)
2. Distribute tasks with `TaskCreate` — all Read the `_workspace/02_*` files from Phase 2
3. Teammates discuss conflicting data via `SendMessage`, reaching consensus through file-based collaboration
4. Generate final integrated document `_workspace/03_integrated.md`
5. Clean up team with `TeamDelete`

### Phase 4: Independent Verification
**Execution Mode:** Sub-Agent

A single QA sub-agent receives `_workspace/03_integrated.md` as input and generates a verification report.
```

**Hybrid transition rules:**
- Team → Sub: Must clean up team with `TeamDelete` before invoking Agent tool
- Sub → Team: Pass sub-agent file artifacts to teammates as Read paths
- Team → Team: Clean up previous team before new `TeamCreate` (only 1 team active per session)

---

## Writing Principles

1. **Specify execution mode first** — State "Agent Team" / "Sub-Agent" / "Hybrid" at the top of the orchestrator. If hybrid, a per-Phase mode table is required
2. **For team mode, specify TeamCreate/SendMessage/TaskCreate usage concretely** — Team formation, task registration, communication rules
3. **For sub mode, fully specify Agent tool parameters** — name, subagent_type, prompt, run_in_background, model
4. **Use absolute file paths** — No relative paths; use clear paths based on `_workspace/`
5. **Specify inter-Phase dependencies** — Which Phase depends on which Phase's results. For hybrid, especially emphasize mode transition points
6. **Keep error handling realistic** — Do not assume "everything will succeed"
7. **Test scenarios are required** — At least 1 normal + 1 error scenario

## Follow-up Task Keywords for description

An orchestrator's description is insufficient with only initial execution keywords. The following follow-up task expressions must be included:

- Re-execute / run again / update / modify / supplement
- "Only redo {part} of {domain}"
- "Based on previous results", "improve results"
- Domain-related everyday requests (e.g., for a launch strategy harness: "launch", "promotion", "trending", etc.)

Without follow-up keywords, the harness effectively becomes dead code after the first execution.

## Orchestrator Reference

Basic structure of a fan-out/fan-in pattern orchestrator:
Preparation → Phase 0 (Context Check) → TeamCreate + TaskCreate → N teammates execute in parallel → Read + Integration → Cleanup.
Refer to the research team example in `references/team-examples.md`.
