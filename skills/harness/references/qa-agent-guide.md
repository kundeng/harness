# QA Agent Design Guide

A reference guide for including a QA agent in a build harness. Based on bug patterns discovered in a real project (SatangSlide) and their root cause analysis, this guide provides a systematic verification methodology for catching defects that QA commonly misses.

---

## Table of Contents

1. Defect Patterns That QA Agents Miss
2. Integration Coherence Verification
3. QA Agent Design Principles
4. Verification Checklist Template
5. QA Agent Definition Template

---

## 1. Defect Patterns That QA Agents Miss

### 1-1. Boundary Mismatch

The most frequent defect. Two components are each implemented "correctly," but the contract breaks at their connection point.

| Boundary | Mismatch Example | Why It's Missed |
|----------|-----------------|-----------------|
| API response → frontend hook | API returns `{ projects: [...] }`, hook expects `SlideProject[]` | Each passes individual verification; no cross-comparison is done |
| API response field name → type definition | API uses `thumbnailUrl` (camelCase), type uses `thumbnail_url` (snake_case) | TypeScript generic casting bypasses the compiler |
| File path → link href | Page is at `/dashboard/create` but link points to `/create` | File structure and href are not cross-compared |
| State transition map → actual status update | Map defines `generating_template → template_approved`, but the code omits this transition | Only checks that the map exists; doesn't trace all update code |
| API endpoint → frontend hook | API exists but has no corresponding hook (never called) | API list and hook list are not mapped 1:1 |
| Immediate response → async result | API immediately returns `{ status }`, frontend accesses `data.failedIndices` | Types are checked without distinguishing sync/async responses |

### 1-2. Why Static Code Review Fails to Catch These

- **Limitations of TypeScript generics**: `fetchJson<SlideProject[]>()` — compiles even if the runtime response is `{ projects: [...] }`
- **Passing `npm run build` ≠ correct behavior**: When type casting, `any`, or generics are used, the build succeeds but fails at runtime
- **Existence check vs. connection check**: "Does the API exist?" and "Does the API's response match the caller's expectations?" are entirely different verifications

---

## 2. Integration Coherence Verification

**Cross-comparison verification** areas that must be included in any QA agent.

### 2-1. API Response ↔ Frontend Hook Type Cross-Verification

**Method**: Compare each API route's `NextResponse.json()` call site with the corresponding hook's `fetchJson<T>` type parameter.

```
Verification steps:
1. Extract the shape of the object passed to NextResponse.json() in the API route
2. Check the T type of fetchJson<T> in the corresponding hook
3. Compare whether the shape and T match
4. Check for wrapping (if the API returns { data: [...] }, does the hook unwrap .data?)
```

**Patterns to watch for:**
- Pagination APIs: `{ items: [], total, page }` vs frontend expecting an array
- Mismatches across snake_case DB fields → camelCase API response → frontend type definitions
- Shape differences between immediate responses (202 Accepted) and final results

### 2-2. File Path ↔ Link/Router Path Mapping

**Method**: Extract URL paths from page files under `src/app/` and compare them against all `href`, `router.push()`, and `redirect()` values in the code.

```
Verification steps:
1. Extract URL patterns from page.tsx file paths under src/app/
   - (group) → removed from URL
   - [param] → dynamic segment
2. Collect all href=, router.push(, redirect( values in the code
3. Verify that each link matches an actually existing page path
4. Watch for URL prefixes of pages inside route groups (e.g., under dashboard/)
```

### 2-3. State Transition Completeness Tracking

**Method**: Extract all `status:` updates from the code and compare them against the state transition map.

```
Verification steps:
1. Extract the list of allowed transitions from the state transition map (STATE_TRANSITIONS)
2. Search all API routes for .update({ status: "..." }) patterns
3. Verify that each transition is defined in the map
4. Identify transitions defined in the map but never executed in code (dead transitions)
5. In particular: check that transitions from intermediate states (e.g., generating_template) to final states (template_approved) are not missing
```

### 2-4. API Endpoint ↔ Frontend Hook 1:1 Mapping

**Method**: List all API routes and frontend hooks to verify they are properly paired.

```
Verification steps:
1. Extract the list of endpoints by HTTP method from route.ts files under src/app/api/
2. Extract the list of fetch call URLs from use*.ts files under src/hooks/
3. Identify API endpoints not called by any hook → flag as "unused"
4. Determine whether "unused" is intentional (e.g., admin APIs) or unintentional (missing call)
```

---

## 3. QA Agent Design Principles

### 3-1. Use a General-Purpose Type, Not an Explore Type

If the QA agent is an `Explore` type, it can only read. But effective QA requires:
- Pattern searching with Grep (extract all `NextResponse.json()` calls)
- Running scripts for automated comparison (API shape vs hook type)
- Making fixes when needed

**Recommended**: Set to `general-purpose` type, but specify a "verify → report → request fix" protocol in the agent definition.

### 3-2. Prioritize "Cross-Comparison" Over "Existence Checks" in Checklists

| Weak Checklist | Strong Checklist |
|----------------|-----------------|
| Does the API endpoint exist? | Does the API endpoint's response shape match the corresponding hook's type? |
| Is the state transition map defined? | Do all status update code paths match the transitions in the map? |
| Does the page file exist? | Do all links in the code point to actually existing pages? |
| Is TypeScript strict mode enabled? | Are there type safety bypasses through generic casting? |

### 3-3. The "Read Both Sides Simultaneously" Principle

To catch boundary bugs, QA must not read only one side. It must:
- Read the API route **and** the corresponding hook **together**
- Read the state transition map **and** the actual update code **together**
- Read the file structure **and** the link paths **together**

State this principle explicitly in the agent definition.

### 3-4. Run QA Immediately After Each Module Is Complete, Not After the Entire Build

If the orchestrator places QA only in "Phase 4: after full completion":
- Bugs accumulate, raising the cost of fixes
- Early boundary mismatches propagate to subsequent modules

**Recommended pattern**: Perform cross-verification of each backend API + its corresponding hook immediately upon API completion (incremental QA).

---

## 4. Verification Checklist Template

An integration coherence checklist for web applications, to be included in the QA agent definition.

```markdown
### Integration Coherence Verification (Web App)

#### API ↔ Frontend Connection
- [ ] Response shapes of all API routes match the generic types of their corresponding hooks
- [ ] Wrapped responses ({ items: [...] }) are unwrapped in hooks
- [ ] snake_case ↔ camelCase conversion is applied consistently
- [ ] Immediate responses (202) and final result shapes are distinguished on the frontend
- [ ] All API endpoints have corresponding frontend hooks that are actually called

#### Routing Coherence
- [ ] All href/router.push values in code match actual page file paths
- [ ] Path verification accounts for route groups ((group)) being removed from URLs
- [ ] Dynamic segments ([id]) are filled with correct parameters

#### State Machine Coherence
- [ ] All defined state transitions are executed in code (no dead transitions)
- [ ] All status updates in code are defined in the transition map (no unauthorized transitions)
- [ ] No missing transitions from intermediate states to final states
- [ ] Values X in frontend state-based branching (if status === "X") are actually reachable

#### Data Flow Coherence
- [ ] Mapping between DB schema field names and API response field names is consistent
- [ ] Frontend type definition field names match API response field names
- [ ] null/undefined handling for optional fields is consistent on both sides
```

---

## 5. QA Agent Definition Template

Core sections to include in the build harness QA agent.

```markdown
---
name: qa-inspector
description: "QA verification specialist. Verifies spec compliance, integration coherence, and design quality."
---

# QA Inspector

## Core Role
Verify implementation quality against specs and **integration coherence across modules**.

## Verification Priority

1. **Integration coherence** (highest) — boundary mismatches are the primary cause of runtime errors
2. **Functional spec compliance** — API/state machine/data model
3. **Design quality** — colors/typography/responsiveness
4. **Code quality** — unused code, naming conventions

## Verification Method: "Read Both Sides Simultaneously"

For boundary verification, always **open and compare code from both sides**:

| Verification Target | Left (Producer) | Right (Consumer) |
|--------------------|-----------------|-----------------|
| API response shape | NextResponse.json() in route.ts | fetchJson<T> in hooks/ |
| Routing | src/app/ page file paths | href, router.push values |
| State transitions | STATE_TRANSITIONS map | .update({ status }) code |
| DB → API → UI | Table column names | API response fields → type definitions |

## Team Communication Protocol

- Upon discovery, send a specific fix request to the responsible agent (file:line + fix method)
- For boundary issues, notify **both** agents on each side
- To the leader: verification report (distinguish passed/failed/unverified items)
```

---

## Real-World Cases: Bugs Found in SatangSlide

All content in this guide is derived from lessons learned from the following actual bugs:

| Bug | Boundary | Cause |
|-----|----------|-------|
| `projects?.filter is not a function` | API→hook | API returns `{projects:[]}`, hook expects an array |
| All dashboard links return 404 | file path→href | Missing `/dashboard/` prefix |
| Theme images not displaying | API→component | `thumbnailUrl` vs `thumbnail_url` |
| Theme selection not saving | API→hook | select-theme API exists, but no hook |
| Creation page waits forever | state transition→code | Missing `template_approved` transition code |
| `data.failedIndices` crash | immediate response→frontend | Accessing background result from immediate response |
| View slides after completion returns 404 | file path→href | `/projects/` → `/dashboard/projects/` |
