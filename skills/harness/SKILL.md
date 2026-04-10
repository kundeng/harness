---
name: harness
description: "Configures the harness. A meta-skill that defines specialized agents and creates the skills those agents will use. Use when: (1) 'configure harness', 'build harness' requests, (2) 'harness design', 'harness engineering' requests, (3) building a harness-based automation system for a new domain/project, (4) restructuring or extending an existing harness configuration, (5) 'harness inspection', 'harness audit', 'harness status', 'agent/skill sync' and other existing harness operations/maintenance requests."
---

# Harness — Agent Team & Skill Architect

A meta-skill that configures a harness tailored to the domain/project, defines each agent's role, and creates the skills those agents will use.

**Core Principles:**
1. Create agent definitions (`.claude/agents/`) and skills (`.claude/skills/`).
2. **Use agent teams as the default execution mode.**
3. **Register harness pointers in CLAUDE.md.** — Record only minimal pointers (trigger rules + change history) so the orchestrator skill is triggered in new sessions.
4. **A harness is not a fixed artifact but an evolving system.** — Incorporate feedback after each execution, and continuously update agents, skills, and CLAUDE.md.

## Workflow

### Phase 0: Status Audit

When the harness skill is triggered, the first step is to check the current harness status.

1. Read `project/.claude/agents/`, `project/.claude/skills/`, `project/CLAUDE.md`
2. Branch execution mode based on current status:
   - **New Build**: Agent/skill directories do not exist or are empty → Execute all phases starting from Phase 1
   - **Existing Extension**: Existing harness present and request to add new agents/skills → Execute only needed phases per the Phase Selection Matrix below
   - **Operations/Maintenance**: Audit, fix, or sync requests for the existing harness → Go to Phase 7-5 Operations/Maintenance Workflow

   **Phase Selection Matrix for Existing Extensions:**
   | Change Type | Phase 1 | Phase 2 | Phase 3 | Phase 4 | Phase 5 | Phase 6 |
   |----------|---------|---------|---------|---------|---------|---------|
   | Add Agent | Skip (use Phase 0 results) | Placement decision only | Required | If dedicated skill needed | Modify orchestrator | Required |
   | Add/Modify Skill | Skip | Skip | Skip | Required | If connections change | Required |
   | Architecture Change | Skip | Required | Affected agents only | Affected skills only | Required | Required |
3. Cross-reference existing agent/skill lists with CLAUDE.md records to detect drift
4. Summarize audit results to the user and confirm the execution plan

### Phase 1: Domain Analysis
1. Identify the domain/project from the user's request
2. Identify core task types (creation, validation, editing, analysis, etc.)
3. Analyze conflicts/overlaps with existing agents/skills based on Phase 0 audit results
4. Explore the project codebase — identify tech stack, data models, key modules
5. **Detect user proficiency** — Gauge technical level from contextual cues in the conversation (terminology used, question sophistication), and adjust communication tone accordingly. For users with limited coding experience, do not use terms like "assertion" or "JSON schema" without explanation.

### Phase 2: Team Architecture Design

#### 2-1. Execution Mode Selection

**Agent teams are the top-priority default.** When 2 or more agents collaborate, always consider agent teams first. Teammates coordinate themselves through direct communication (SendMessage) and shared task lists (TaskCreate), and discovery sharing, conflict discussion, and gap-filling improve result quality.

| Mode | When to Use | Characteristics |
|------|----------|------|
| **Agent Team** (default) | 2+ collaborators, real-time coordination and feedback exchange needed, intermediate artifacts cross-referenced | Self-coordinating via `TeamCreate` + `SendMessage` + `TaskCreate` |
| **Sub-Agent** (alternative) | Single-agent tasks, returning results to main is sufficient, team communication overhead is excessive | Direct `Agent` tool calls, parallel via `run_in_background` |
| **Hybrid** | When each phase has different characteristics — e.g., parallel collection (sub) → consensus-based integration (team) | Mix team/sub per phase |

**Decision Order:**
1. First check if the design works as an agent team — default when 2+ agents
2. Select sub-agent only when team communication is structurally unnecessary (result passing only) and team overhead outweighs the benefits
3. Consider hybrid when phase characteristics are distinctly different — specify the execution mode for each phase in the orchestrator

> For detailed comparison tables and pattern-specific decision trees, see "Execution Modes" in `references/agent-design-patterns.md`.

#### 2-2. Architecture Pattern Selection

1. Decompose work into specialized domains
2. Determine agent team structure (see `references/agent-design-patterns.md` for architecture patterns)
   - **Pipeline**: Sequential dependent tasks
   - **Fan-out/Fan-in**: Parallel independent tasks
   - **Expert Pool**: Situational selective invocation
   - **Producer-Reviewer**: Quality review after creation
   - **Supervisor**: Central agent manages state and dynamic distribution
   - **Hierarchical Delegation**: Upper-level agents recursively delegate to lower-level agents

#### 2-3. Agent Separation Criteria

Evaluate along 4 axes: specialization, parallelism, context, and reusability. For the detailed criteria table, see "Agent Separation Criteria" in `references/agent-design-patterns.md`.

### Phase 3: Agent Definition Creation

**All agents must be defined as `project/.claude/agents/{name}.md` files.** Placing roles directly in the Agent tool's prompt without an agent definition file is prohibited. Reasons:
- Agent definitions must exist as files to be reusable across sessions
- Team communication protocols must be specified to ensure collaboration quality between agents
- The core value of the harness is the separation of agents (who) and skills (how)

Even when using built-in types (`general-purpose`, `Explore`, `Plan`), create agent definition files. Specify built-in types via the `subagent_type` parameter of the Agent tool, and include roles, principles, and protocols in the agent definition file.

**Model Setting:** All agents use `model: "opus"`. Always specify the `model: "opus"` parameter when calling the Agent tool. Harness quality is directly tied to agent reasoning capability, and opus guarantees the highest quality.

**Team Reconfiguration:** Only one team can be active per session, but teams can be disbanded and reconstituted between phases. When different specialist combinations are needed per phase (as in the pipeline pattern), save the previous team's artifacts to files, clean up the team, and create a new team.

Define each agent in `project/.claude/agents/{name}.md`. Required sections: core role, working principles, input/output protocol, error handling, collaboration. In agent team mode, add a `## Team Communication Protocol` section specifying message receive/send targets and task request scope.

> For definition templates and full example files, see "Agent Definition Structure" in `references/agent-design-patterns.md` + `references/team-examples.md`.

**Requirements When Including a QA Agent:**
- QA agents must use the `general-purpose` type (`Explore` is read-only and cannot run validation scripts)
- The core of QA is not "existence checking" but **"cross-boundary comparison"** — simultaneously reading API responses and front-end hooks to compare shapes
- QA should not run once after full completion, but **incrementally right after each module is completed** (incremental QA)
- Detailed guide: see `references/qa-agent-guide.md`

### Phase 4: Skill Creation

Create skills for each agent at `project/.claude/skills/{name}/SKILL.md`. For detailed writing guidelines, see `references/skill-writing-guide.md`.

#### 4-1. Skill Structure

```
skill-name/
├── SKILL.md (required)
│   ├── YAML frontmatter (name, description required)
│   └── Markdown body
└── Bundled Resources (optional)
    ├── scripts/    - Executable code for repetitive/deterministic tasks
    ├── references/ - Reference documents loaded conditionally
    └── assets/     - Files used in output (templates, images, etc.)
```

#### 4-2. Writing Descriptions — Aggressive Trigger Induction

The description is the skill's sole trigger mechanism. Since Claude tends to judge triggers conservatively, write descriptions **aggressively ("pushy")**.

**Bad example:** `"A skill that processes PDF documents"`
**Good example:** `"Performs all PDF operations including reading PDF files, extracting text/tables, merging, splitting, rotating, watermarking, encrypting, OCR, etc. Always use this skill when .pdf files are mentioned or PDF output is requested."`

Key: Describe everything the skill does + specific trigger situations, and differentiate from similar cases that should NOT trigger.

#### 4-3. Body Writing Principles

| Principle | Description |
|------|------|
| **Explain the Why** | Instead of coercive directives like "ALWAYS/NEVER", convey the reason why something should be done. When LLMs understand the reason, they make correct judgments even in edge cases. |
| **Keep it Lean** | The context window is a shared resource. Target under 500 lines for the SKILL.md body; delete content that doesn't carry its weight or move it to references/. |
| **Generalize** | Rather than narrow rules that fit only specific examples, explain principles to handle diverse inputs. No overfitting. |
| **Bundle Repetitive Code** | When test runs reveal scripts that agents commonly write, pre-bundle them in `scripts/`. |
| **Write in Imperative Form** | Use imperative/directive tone in the form of commands and instructions. |

#### 4-4. Progressive Disclosure

Skills manage context through a 3-tier loading system:

| Tier | When Loaded | Size Target |
|------|----------|----------|
| **Metadata** (name + description) | Always present in context | ~100 words |
| **SKILL.md body** | When skill is triggered | <500 lines |
| **references/** | Only when needed | Unlimited (scripts can be executed without loading) |

**Size Management Rules:**
- When SKILL.md approaches 500 lines, split detailed content into references/ and leave a pointer in the body indicating "when to read this file"
- Include a **table of contents (ToC)** at the top of reference files over 300 lines
- When domain/framework-specific variations exist, separate them under references/ by domain so only relevant files are loaded

```
cloud-deploy/
├── SKILL.md (workflow + selection guide)
└── references/
    ├── aws.md    ← Load only when AWS is selected
    ├── gcp.md
    └── azure.md
```

#### 4-5. Skill-Agent Linking Principles

- 1 agent ↔ 1~N skills (1:1 or 1:many)
- Skills shared by multiple agents are also possible
- Skills contain "how to do it", agents contain "who does it"

> For detailed writing patterns, examples, and data schema standards, see `references/skill-writing-guide.md`.

### Phase 5: Integration and Orchestration

The orchestrator is a specialized form of skill that weaves individual agents and skills into a single workflow, coordinating the entire team. If the individual skills created in Phase 4 define "what each agent does and how", the orchestrator defines "who collaborates in what order and when". For concrete templates, see `references/orchestrator-template.md`.

**Modifying the Orchestrator for Existing Extensions:** When extending an existing harness rather than building from scratch, modify the existing orchestrator instead of creating a new one. When adding agents, reflect the new agents in team composition, task assignment, and data flow, and add trigger keywords related to the new agents in the description.

The orchestrator pattern varies depending on the execution mode selected in Phase 2-1:

#### 5-0. Orchestrator Patterns (by Mode)

**Agent Team Pattern (default):**
The orchestrator assembles the team with `TeamCreate` and assigns tasks with `TaskCreate`. Teammates communicate directly via `SendMessage` for self-coordination. The leader (orchestrator) monitors progress and synthesizes results.

```
[Orchestrator/Leader]
    ├── TeamCreate(team_name, members)
    ├── TaskCreate(tasks with dependencies)
    ├── Teammates self-coordinate (SendMessage)
    ├── Collect and synthesize results
    └── Clean up team
```

**Sub-Agent Pattern (alternative):**
The orchestrator directly calls sub-agents using the `Agent` tool. Parallel execution uses `run_in_background: true`, and results are returned only to the main agent. Use when team communication is unnecessary and you want to reduce overhead.

```
[Orchestrator]
    ├── Agent(agent-1, run_in_background=true)
    ├── Agent(agent-2, run_in_background=true)
    ├── Wait for and collect results
    └── Generate integrated artifacts
```

**Hybrid Pattern:**
Mix different modes per phase. Common combinations:
- **Parallel collection (sub) → Consensus integration (team)**: Phase 2 uses sub-agents for parallel independent data collection → Phase 3 creates a team for discussion and consensus-based integration
- **Team creation (team) → Validation (sub)**: Phase 2 has the team create a draft → Phase 3 has a single sub-agent perform independent validation
- **Team reconfiguration between phases**: `TeamDelete` then new `TeamCreate` for each phase, with sub-agent calls inserted in between

When choosing hybrid, specify the execution mode at the top of each phase section in the orchestrator (e.g., `**Execution Mode:** Agent Team`).

#### 5-1. Data Passing Protocol

Specify the inter-agent data passing method within the orchestrator:

| Strategy | Method | Applicable Mode | Suitable When |
|------|------|----------|-----------|
| **Message-based** | Direct communication between teammates via `SendMessage` | Team | Real-time coordination, feedback exchange, lightweight state passing |
| **Task-based** | Share task status via `TaskCreate`/`TaskUpdate` | Team | Progress tracking, dependency management, task requests |
| **File-based** | Write and read files at agreed-upon paths | Team + Sub | Large data, structured artifacts, audit trail needed |
| **Return-value-based** | Return message from the `Agent` tool | Sub | Main agent directly collects sub-agent results |

**Recommended combination (team mode):** Task-based (coordination) + File-based (artifacts) + Message-based (real-time communication)
**Recommended combination (sub mode):** Return-value-based (result collection) + File-based (large artifacts)
**Hybrid:** Apply the appropriate combination for each phase's execution mode

Rules for file-based passing:
- Create a `_workspace/` folder under the working directory to store intermediate artifacts
- File naming convention: `{phase}_{agent}_{artifact}.{ext}` (e.g., `01_analyst_requirements.md`)
- Output only final artifacts to user-specified paths; preserve intermediate files (`_workspace/`) for post-verification and audit trail

#### 5-2. Error Handling

Include error handling policies within the orchestrator. Core principle: retry once, and if it fails again, proceed without that result (note the omission in the report); do not delete conflicting data, but cite sources.

> For error type-specific strategy tables and implementation details, see "Error Handling" in `references/orchestrator-template.md`.

#### 5-3. Team Size Guidelines

| Task Scale | Recommended Team Size | Tasks per Teammate |
|----------|------------|--------------|
| Small (5-10 tasks) | 2-3 | 3-5 |
| Medium (10-20 tasks) | 3-5 | 4-6 |
| Large (20+ tasks) | 5-7 | 4-5 |

> The more teammates, the greater the coordination overhead. 3 focused teammates are better than 5 unfocused ones.

#### 5-4. Registering Harness Pointers in CLAUDE.md

After harness configuration is complete, register minimal pointers in the project's `CLAUDE.md`. Since CLAUDE.md is loaded with every new session, recording only the harness existence and trigger rules is sufficient — the orchestrator skill handles the rest.

**CLAUDE.md Template:**

````markdown
## Harness: {Domain Name}

**Goal:** {One-line core goal of the harness}

**Trigger:** Use the `{orchestrator-skill-name}` skill when {domain}-related task requests are made. Simple questions can be answered directly.

**Change History:**
| Date | Change Description | Target | Reason |
|------|----------|------|------|
| {YYYY-MM-DD} | Initial configuration | All | - |
````

**What NOT to put in CLAUDE.md:** Agent lists, skill lists, directory structure, detailed execution rules. Reason: Agent/skill lists are managed by the orchestrator skill and `.claude/agents/`, `.claude/skills/`, so it would be redundant. Directory structure can be verified directly from the file system. CLAUDE.md holds **only pointers (trigger rules) + change history**.

#### 5-5. Follow-up Task Support

The orchestrator must handle not only initial execution but also follow-up tasks. Ensure the following three things:

**1. Include follow-up keywords in the orchestrator description:**
Initial creation keywords alone will not trigger follow-up requests. Expressions that must be included in the description:
- "re-run", "rerun", "update", "modify", "supplement"
- "redo only {subtask} of {domain}"
- "based on previous results", "improve results"

**2. Add a context check step to orchestrator Phase 1:**
Check for existing artifacts at the start of the workflow to determine the execution mode:
- `_workspace/` exists + user requests partial modification → **Partial re-execution** (re-invoke only the relevant agent)
- `_workspace/` exists + user provides new input → **New execution** (move existing _workspace to `_workspace_prev/`)
- `_workspace/` does not exist → **Initial execution**

**3. Include re-invocation instructions in agent definitions:**
Specify "behavior when previous artifacts exist" in each agent `.md` file:
- If previous result files exist, read them and incorporate improvements
- If user feedback is provided, modify only the relevant parts

> See the "Phase 0: Context Check" section in the orchestrator template: `references/orchestrator-template.md`

### Phase 6: Validation and Testing

Validate the generated harness. For detailed testing methodology, see `references/skill-testing-guide.md`.

#### 6-1. Structural Validation

- Verify all agent files are in the correct locations
- Validate skill frontmatter (name, description)
- Check referential consistency between agents
- Verify no commands were created

#### 6-2. Execution Mode-Specific Validation

- **Agent Team**: Verify communication paths between teammates, task dependencies, team size appropriateness
- **Sub-Agent**: Verify I/O connections for each agent, `run_in_background` settings, return value collection logic
- **Hybrid**: Verify execution modes are specified for each phase in the orchestrator, and data passing is not broken at phase boundaries (when transitioning from team → sub, verify the team's artifacts connect as sub-agent inputs)

#### 6-3. Skill Execution Testing

Perform actual execution tests for each generated skill:

1. **Write test prompts** — Write 2-3 realistic test prompts for each skill. Use specific, natural sentences that a real user would actually type.

2. **With-skill vs Without-skill comparison runs** — When possible, run with-skill and without-skill executions in parallel to confirm the skill's added value. Spawn two agents each:
   - **With-skill**: Read the skill and perform the task
   - **Without-skill (baseline)**: Perform the same prompt without the skill

3. **Evaluate results** — Assess artifact quality through qualitative (user review) + quantitative (assertion-based) evaluation. Define assertions when artifacts are objectively verifiable (file creation, data extraction, etc.), and rely on user feedback when subjective (tone, design).

4. **Iterative improvement loop** — When issues are found in test results:
   - **Generalize** the feedback to modify the skill (no narrow fixes that only address specific examples)
   - Re-test after modification
   - Repeat until the user is satisfied or no meaningful improvements remain

5. **Bundle repetitive patterns** — When test runs reveal code that agents commonly write (e.g., the same helper script generated in every test), pre-bundle that code in `scripts/`.

#### 6-4. Trigger Validation

Verify that each skill's description triggers correctly:

1. **Should-trigger queries** (8-10) — Various expressions that should trigger the skill (formal/casual, explicit/implicit)
2. **Should-NOT-trigger queries** (8-10) — "Near-miss" queries with similar keywords but where a different tool/skill is appropriate

**Near-miss writing key:** Obviously unrelated queries like "write a Fibonacci function" have no testing value. Queries with **ambiguous boundaries** like "extract charts from this Excel file as PNG" (xlsx skill vs image conversion) make good test cases.

Also check for trigger conflicts with existing skills at this stage.

#### 6-5. Dry Run Testing

- Review whether the orchestrator skill's phase order is logical
- Verify there are no dead links in data passing paths
- Confirm all agents' inputs match the outputs from previous phases
- Verify fallback paths for each error scenario are executable

#### 6-6. Test Scenario Writing

- Add a `## Test Scenarios` section to the orchestrator skill
- Document at least 1 normal flow + 1 error flow

### Phase 7: Harness Evolution

A harness is not a static artifact built once and done. It is a system that continuously evolves based on user feedback.

#### 7-1. Post-Execution Feedback Collection

After each harness execution, request feedback from the user:
- "Is there anything to improve in the results?"
- "Is there anything you'd like to change about the agent team composition or workflow?"

If there is no feedback, move on. Do not force it, but always provide the opportunity.

#### 7-2. Feedback Incorporation Paths

The modification target varies by feedback type:

| Feedback Type | Modification Target | Example |
|-----------|----------|------|
| Output quality | The relevant agent's skill | "Analysis is too superficial" → Add depth criteria to skill |
| Agent role | Agent definition `.md` | "Security review is also needed" → Add new agent |
| Workflow order | Orchestrator skill | "Validation should come first" → Change phase order |
| Team composition | Orchestrator + agents | "These two could be merged" → Merge agents |
| Missing trigger | Skill description | "It doesn't work with this expression" → Expand description |

#### 7-3. Change History

All changes are recorded in the **Change History** table in CLAUDE.md (same table as the "Change History" section in the Phase 5-4 template):

```markdown
**Change History:**
| Date | Change Description | Target | Reason |
|------|----------|------|------|
| 2026-04-05 | Initial configuration | All | - |
| 2026-04-07 | Added QA agent | agents/qa.md | Feedback: insufficient output quality validation |
| 2026-04-10 | Added tone guide | skills/content-creator | Feedback: "too stiff" |
```

This history tracks the direction of harness evolution and prevents regression.

#### 7-4. Evolution Triggers

Not only when the user explicitly says "modify the harness", but also suggest evolution in the following situations:
- When the same type of feedback repeats 2 or more times
- When a pattern of repeated agent failures is discovered
- When the user is observed bypassing the orchestrator to work manually

#### 7-5. Operations/Maintenance Workflow

Systematically perform inspection, modification, and synchronization of the existing harness. Follow this workflow when entering the "Operations/Maintenance" branch from Phase 0.

**Step 1: Status Audit**
- Compare `.claude/agents/` file list with the orchestrator skill's agent configuration → Generate discrepancy list
- Compare `.claude/skills/` directory list with the orchestrator skill's skill configuration → Generate discrepancy list
- Report audit results to the user

**Step 2: Incremental Addition/Modification**
- Add/modify/delete agents and skills per user request
- Make changes one at a time; run Step 3 (synchronization) immediately after each change

**Step 3: Update CLAUDE.md Change History**
- Record the date, change description, target, and reason in the change history table

**Step 4: Change Validation**
- Structural validation of modified agents/skills (per Phase 6-1 criteria)
- Trigger validation if the modification scope affects triggers (per Phase 6-4 criteria)
- For large-scale changes (architecture changes, 3+ agents added/deleted), also perform Phase 6-3 (execution testing) and 6-5 (dry run)
- Final verification of consistency between CLAUDE.md and actual files

## Artifact Checklist

Verify after creation is complete:

- [ ] `project/.claude/agents/` — **Agent definition files must be created** (file creation required even for built-in types)
- [ ] `project/.claude/skills/` — Skill files (SKILL.md + references/)
- [ ] 1 orchestrator skill (including data flow + error handling + test scenarios)
- [ ] Execution mode specified (agent team / sub-agent / hybrid; if hybrid, mode per phase noted)
- [ ] `model: "opus"` parameter specified in all Agent calls
- [ ] `.claude/commands/` — Nothing created
- [ ] No conflicts with existing agents/skills
- [ ] Skill descriptions are written aggressively ("pushy") — **follow-up task keywords included**
- [ ] SKILL.md body is under 500 lines; split to references/ if exceeded
- [ ] Execution verified with 2-3 test prompts
- [ ] Trigger validation (should-trigger + should-NOT-trigger) completed
- [ ] **Harness pointer registered in CLAUDE.md** (trigger rules + change history)
- [ ] **Agent/skill additions/deletions/modifications recorded in CLAUDE.md change history**
- [ ] **Context check step in orchestrator Phase 1** (determine initial/follow-up/partial re-execution)

## References

- Harness patterns: `references/agent-design-patterns.md`
- Existing harness examples (including full actual files): `references/team-examples.md`
- Orchestrator template: `references/orchestrator-template.md`
- **Skill writing guide**: `references/skill-writing-guide.md` — Writing patterns, examples, data schema standards
- **Skill testing guide**: `references/skill-testing-guide.md` — Testing/evaluation/iterative improvement methodology
- **QA agent guide**: `references/qa-agent-guide.md` — Reference when including a QA agent in a build harness. Includes integration consistency validation methodology, boundary bug patterns, and QA agent definition templates. Based on 7 bug cases discovered in actual projects.
