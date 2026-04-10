# Skill Testing & Iterative Improvement Guide

A methodology for validating and iteratively improving the quality of skills created in the harness. A supplementary reference for SKILL.md Phase 6.

---

## Table of Contents

1. [Testing Framework Overview](#1-testing-framework-overview)
2. [Writing Test Prompts](#2-writing-test-prompts)
3. [Execution Testing: With-skill vs Baseline](#3-execution-testing-with-skill-vs-baseline)
4. [Quantitative Evaluation: Assertion-Based Scoring](#4-quantitative-evaluation-assertion-based-scoring)
5. [Leveraging Specialized Agents](#5-leveraging-specialized-agents)
6. [Iterative Improvement Loop](#6-iterative-improvement-loop)
7. [Description Trigger Verification](#7-description-trigger-verification)
8. [Workspace Structure](#8-workspace-structure)

---

## 1. Testing Framework Overview

Skill quality verification is a combination of **qualitative evaluation** and **quantitative evaluation**.

| Evaluation Type | Method | Suitable Skills |
|----------|------|-----------|
| **Qualitative** | User directly reviews the output | Writing style, design, creative works, and other subjective quality aspects |
| **Quantitative** | Assertion-based automated scoring | File generation, data extraction, code generation, and other objectively verifiable tasks |

Core loop: **Write → Run Tests → Evaluate → Improve → Retest**

---

## 2. Writing Test Prompts

### Principles

Test prompts should be **specific, natural sentences that a real user would actually type**. Abstract or artificial prompts have low testing value.

### Bad Examples

```
"Process the PDF"
"Extract the data"
"Generate a chart"
```

### Good Examples

```
"From 'Q4_Sales_Final_v2.xlsx' in the Downloads folder, use column C (revenue)
and column D (costs) to add a profit margin (%) column. Then sort by profit
margin in descending order."
```

```
"Extract the table on page 3 of this PDF and convert it to CSV. The table header
spans 2 rows — the first row is the category, and the second row contains the
actual column names."
```

### Prompt Diversity

- Mix **formal / casual** tones
- Mix **explicit / implicit** intent (cases where the file format is stated directly vs. cases where it must be inferred from context)
- Mix **simple / complex** tasks
- Include some abbreviations, typos, and casual expressions

### Coverage

Start with 2-3 prompts, designed to cover:
- 1 core use case
- 1 edge case
- (Optional) 1 compound task

---

## 3. Execution Testing: With-skill vs Baseline

### 3-1. Comparative Execution Structure

For each test prompt, spawn two sub-agents **simultaneously**:

**With-skill execution:**
```
Prompt: "{test prompt}"
Skill path: {skill path}
Output path: _workspace/iteration-N/eval-{id}/with_skill/outputs/
```

**Baseline execution:**
```
Prompt: "{test prompt}"  (identical)
Skill: none
Output path: _workspace/iteration-N/eval-{id}/without_skill/outputs/
```

### 3-2. Baseline Selection

| Scenario | Baseline |
|------|----------|
| New skill creation | Run the same prompt without the skill |
| Existing skill improvement | Previous skill version (preserve snapshot) |

### 3-3. Timing Data Capture

**Immediately** save `total_tokens` and `duration_ms` from the sub-agent completion notification. This data is only accessible at the time of notification and cannot be recovered later.

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

---

## 4. Quantitative Evaluation: Assertion-Based Scoring

### 4-1. Writing Assertions

When outputs are objectively verifiable, define assertions for automated scoring.

**Good assertions:**
- Can be objectively determined as true/false
- Have descriptive names so the purpose is clear just from looking at the results
- Verify the core value of the skill

**Bad assertions:**
- Always pass regardless of whether the skill is used (e.g., "output exists")
- Require subjective judgment (e.g., "well written")

### 4-2. Programmable Verification

If an assertion can be verified with code, write it as a script. This is faster and more reliable than visual inspection, and can be reused across iterations.

### 4-3. Beware of Non-Discriminating Assertions

Assertions that "pass at 100% in both configurations" do not measure the differentiating value of the skill. If you find such assertions, remove them or replace them with more challenging ones.

### 4-4. Scoring Result Schema

```json
{
  "expectations": [
    {
      "text": "Profit margin column added",
      "passed": true,
      "evidence": "Confirmed 'profit_margin_pct' column in column E"
    },
    {
      "text": "Sorted in descending order by profit margin",
      "passed": false,
      "evidence": "Original order maintained without sorting"
    }
  ],
  "summary": {
    "passed": 1,
    "failed": 1,
    "total": 2,
    "pass_rate": 0.50
  }
}
```

---

## 5. Leveraging Specialized Agents

Using agents with specialized roles during the testing/evaluation process improves quality.

### 5-1. Grader

Performs assertion-based scoring and extracts verifiable claims from the output for cross-validation.

**Role:**
- Pass/fail determination per assertion + providing evidence
- Extracting and verifying factual claims from outputs
- Feedback on the quality of the eval itself (suggestions when assertions are too easy or ambiguous)

### 5-2. Comparator (Blind Comparator)

Anonymizes the two outputs as A/B and judges quality without knowing which one used the skill.

**When to use:** When you want to rigorously confirm "Is the new version really better?" Can be skipped during routine iterative improvement.

**Judgment criteria:**
- Content: accuracy, completeness
- Structure: organization, formatting, usability
- Overall score

### 5-3. Analyzer

Analyzes statistical patterns from benchmark data:
- Non-discriminating assertions (both configurations pass → no differentiating power)
- High-variance evals (results vary significantly across runs → unstable)
- Time/token tradeoffs (when the skill improves quality but also increases cost)

---

## 6. Iterative Improvement Loop

### 6-1. Collecting Feedback

Show the output to the user and collect feedback. Empty feedback is interpreted as "no issues."

### 6-2. Improvement Principles

1. **Generalize the feedback** — Narrow fixes that only work for the test example are overfitting. Make corrections at the principle level.
2. **Remove anything that doesn't pull its weight** — Read the transcript, and if the skill is making the agent do unproductive work, delete that part.
3. **Explain the why** — Even if the user's feedback is brief, understand why it matters and reflect that understanding in the skill.
4. **Bundle repetitive tasks** — If the same helper script is generated in every test run, pre-include it in `scripts/`.

### 6-3. Iteration Procedure

```
1. Modify the skill
2. Re-run all test cases in a new iteration-N+1/ directory
3. Present results to the user (compare with previous iteration)
4. Collect feedback
5. Modify again → repeat
```

**Termination conditions:**
- User is satisfied
- All feedback is empty (no issues with any output)
- No further meaningful improvements possible

### 6-4. Draft → Review Pattern

When modifying a skill, write a draft first, then **re-read it with fresh eyes** and improve. Don't try to write it perfectly in one pass — go through a draft-review cycle.

---

## 7. Description Trigger Verification

### 7-1. Writing Trigger Eval Queries

Write 20 eval queries — 10 should-trigger + 10 should-NOT-trigger.

**Query quality criteria:**
- Specific, natural sentences that a real user would actually type
- Include concrete details such as file paths, personal context, column names, company names, etc.
- Mix of length, tone, and format
- Focus on **edge cases** rather than clear-cut answers

**Should-trigger queries (8-10):**
- Same intent expressed in various ways (formal/casual)
- Cases where the skill/file type is not explicitly mentioned but clearly needed
- Non-mainstream use cases
- Cases that compete with other skills but this skill should win

**Should-NOT-trigger queries (8-10):**
- **Near-misses are key** — queries with similar keywords but where a different tool/skill is more appropriate
- Obviously unrelated queries ("write a Fibonacci function") have no testing value
- Adjacent domains, ambiguous wording, keyword overlap but different context

### 7-2. Existing Skill Conflict Verification

Verify that the new skill's description does not overlap with the trigger areas of existing skills:

1. Collect the descriptions of existing skills
2. Confirm that the new skill's should-trigger queries do not incorrectly trigger existing skills
3. If conflicts are found, describe the boundary conditions in the description more clearly

### 7-3. Automated Optimization (Optional Advanced Feature)

When description optimization is needed:

1. Split the 20 eval queries into Train (60%) / Test (40%)
2. Measure trigger accuracy with the current description
3. Analyze failure cases and generate an improved description
4. Select the best description based on the Test set (not the Train set — to prevent overfitting)
5. Repeat up to 5 times

> This process is performed with an automation script using `claude -p`. Since token costs are high, run it as a final step after the skill has been sufficiently stabilized.

---

## 8. Workspace Structure

A directory structure for systematically managing test/evaluation results:

```
{skill-name}-workspace/
├── iteration-1/
│   ├── eval-descriptive-name-1/
│   │   ├── eval_metadata.json
│   │   ├── with_skill/
│   │   │   ├── outputs/
│   │   │   ├── timing.json
│   │   │   └── grading.json
│   │   └── without_skill/
│   │       ├── outputs/
│   │       ├── timing.json
│   │       └── grading.json
│   ├── eval-descriptive-name-2/
│   │   └── ...
│   └── benchmark.json
├── iteration-2/
│   └── ...
└── evals/
    └── evals.json
```

**Rules:**
- Eval directories use **descriptive names**, not numbers (e.g., `eval-multi-page-table-extraction`)
- Each iteration is preserved in an independent directory (do not overwrite previous iterations)
- `_workspace/` is never deleted — it serves as a post-hoc verification and audit trail
