---
name: design-to-beads
description: Use when converting a design document, PRD, or task list into beads issues - ensures lossless conversion with proper epic hierarchy, validated dependencies for maximum parallelization, and three independent subagent review passes before execution
---

# Design to Beads Conversion

## Overview

Convert design documents into fully-structured beads issues with **lossless information transfer**, proper epic hierarchy, and validated dependencies.

**Core principle:** Every piece of actionable content in the source document must map to a beads issue. Nothing gets lost. Dependencies must enable maximum parallelization. Three independent subagent reviews catch what self-review misses.

## When to Use

- Converting finalized design docs to trackable work
- Breaking down PRDs into implementation tasks
- Creating beads structure from any markdown plan

## When NOT to Use

- Design isn't finalized (use brainstorming skill first)
- Simple single-task work (just create issue directly)

## Bead Sizing Constraints

Builder agents (Sonnet) have a 200K token context window. Each bead must be completable within **60% (~120K tokens)** of that budget. After fixed overhead (~35-50K tokens for agent prompt, CLAUDE.md, bd metadata, tool call structure, git operations), roughly **70-85K tokens remain** for codebase exploration, implementation, and test runs.

### Hard Limits Per Bead

| Metric | Limit | Why |
|--------|-------|-----|
| Files to create/modify | **≤5** | Each file read+edit cycle costs 1-3K tokens |
| Acceptance criteria | **≤6** | Each AC adds exploration + implementation + test verification |
| Files requiring exploration (context reads) | **≤10** | Each file read costs 500-2K tokens |
| Description length | **≤300 words** | Longer descriptions signal the bead needs decomposition |
| Module/domain scope | **1 domain** | Cross-domain beads require exploring too many patterns |

### Decomposition Triggers

If ANY of these are true, the item MUST be split into smaller beads:

1. **Full-stack slices**: "Add API endpoint + service + schema + frontend component + hook + page" → split into backend bead and frontend bead (frontend depends on backend)
2. **Multiple CRUD operations**: "Implement CRUD for resource X" → split into Create, Read, Update, Delete (or group logically: Create+Read, Update+Delete)
3. **Cross-module work**: "Wire up module A to module B and add module C" → one bead per module boundary
4. **>6 acceptance criteria**: Split by functional grouping — each group becomes its own bead
5. **>5 files to modify**: Group changes by file proximity — files in the same directory become one bead

### Decomposition Strategy

When splitting an oversized item:
1. **Identify the natural seams** — API boundary, module boundary, data flow boundary
2. **Create the producer bead first** (the one that creates the interface/data other beads consume)
3. **Add dependency edges**: consumer bead `depends_on` producer bead
4. **Preserve all detail** — splitting is not summarizing. Each sub-bead gets the full detail from its portion of the source
5. **Name clearly**: "Phase 2.1a: Stripe webhook handler (backend)" not "Phase 2.1 part 1"

### Sizing Column in Coverage Matrix

Add an `Est. Size` column to flag oversized beads during Phase 2 (before review passes):

```
Source Section → Beads Location        | Est. Files | Est. ACs | Flag
────────────────────────────────────────|────────────|──────────|──────
"Phase 1.1 Stripe Setup"  → Issue 1.1  | 3          | 4        | ✓ OK
"Phase 2 Full Integration" → Issue 2.1 | 8          | 10       | ⚠️ SPLIT
"Phase 3 Cleanup"          → Issue 3.1 | 2          | 2        | ✓ OK
```

Any issue flagged `⚠️ SPLIT` must be decomposed before proceeding to review passes.

## Test Architecture

**Why this matters:** Without an explicit test structure, each bead independently creates its own test file. After 10 beads you have 10 fragmented files covering only their own narrow slices, no end-to-end validation that the epic actually works, and unit-level tests that miss integration bugs (auth flow + DB write + UI render) where the failure is in the seam, not the part. This skill bakes test structure into the epic so confidence scales with the epic — not just the bead.

### Three rules

**1. Tests follow the codebase, not the epic.** Test files are owned by the longest-lived structural unit they cover:
- **Unit tests** live with the source module — typically `<module>.test.<ext>` colocated with `<module>.<ext>`, or wherever the codebase already puts them. A bead extends the test file of the module it modifies. Multiple epics that touch the same module extend the *same* file.
- **Integration tests** live in a feature-domain file — `<domain>.integration.test.<ext>` (e.g., `auth.integration.test.ts` covers login, signup, password reset). Multiple epics in the same domain extend the same integration file.

Beads never create new test files when a file already exists for the module or domain. This is what scales: at 100 epics, the test surface tracks the codebase's ~20–40 modules and ~10–20 feature domains, not the bead count.

**2. Mandatory integration-test bead per epic.** Every epic with **≥3 implementation beads** MUST include a final integration-test bead whose job is to validate the epic's stated outcome end-to-end. This bead `depends_on` every sibling implementation bead. Its acceptance criteria reference the epic's user-facing flow, not individual functions. The bead's *cases* live in the domain's shared integration file — but the bead itself stays per-epic so failures point at a specific outcome.

**3. Test Plan field on every bead.** Every bead's description includes a required `Test plan:` line in this exact form:

```
Test plan: Adds N cases to <test-file>. Covers: <behavior 1>; <behavior 2>; <behavior 3>.
```

The named file must already exist in the Test File Plan (Phase 2) — either a module's unit file or a domain's integration file. This forces the builder to commit upfront and prevents the "I'll just create a new test file" reflex. For the integration-test bead, the plan reads `Adds N cases to <domain>.integration.test.<ext>. Covers: <user flow 1>; <user flow 2>.`

### What the integration-test bead looks like

The integration-test bead is a real bead, not a checkbox. It has:
- **Title:** `Epic <N> integration tests: <epic outcome>` (e.g., "Epic 2 integration tests: webhook delivery end-to-end")
- **Description:** What user-facing flows the epic enables, and how the test exercises them
- **Acceptance criteria:** 3–5 ACs each describing a complete flow (not "function X returns Y" — that's a unit bead's job)
- **Dependencies:** `depends_on` every sibling implementation bead
- **Test plan:** `Adds N cases to <domain>.integration.test.<ext>. Covers: <flow 1>; <flow 2>; <flow 3>.`

This bead carries the epic's confidence. If it passes, the epic is shippable. If it fails, the integration is broken regardless of unit test results.

### Single-bead epic exemption

Epics with ≤2 implementation beads MAY skip the integration-test bead — there's nothing meaningful to integrate. Note the exemption explicitly in the Test File Plan (see Phase 2). Their unit tests still go in the relevant module's existing test file, never a per-bead or per-epic file.

## Critical Rules

**DO NOT rush under time pressure.** Quality conversion takes time. A poorly-structured beads hierarchy causes more delay than taking 30 extra minutes upfront.

**DO NOT skip review passes.** Single-pass creation misses gaps, creates false dependencies, loses information.

**DO NOT self-review.** Launch subagents for review passes. Self-review misses what independent reviewers catch.

**DO NOT create flat issue lists.** Use epics with parent-child relationships.

**DO NOT assume sequential ordering.** Question every dependency: "Does A truly BLOCK B, or could they run in parallel?"

**DO NOT create sparse issues.** If the source has detail, the issue must have detail. Brief descriptions = lost information = implementation failures.

**DO NOT create oversized beads.** A bead that exceeds the sizing limits will exhaust the builder agent's context window and fail. Split it.

**DO NOT create new test files when one already exists for the module or domain.** Unit tests follow the source module — beads extend the existing `<module>.test.<ext>`. Integration tests follow the feature domain — beads extend the existing `<domain>.integration.test.<ext>`. New files per bead, or per epic, fragment coverage.

**DO NOT skip the integration-test bead.** Every epic with ≥3 implementation beads requires a dedicated integration-test bead that depends on all siblings. Unit tests alone don't prove the epic works end-to-end.

**DO NOT omit the Test Plan field.** Every bead description must declare its target test file and the cases it adds. Without this, builders default to creating a new test file per bead.

**DO NOT skip the structured Test Plan metadata.** The prose `Test plan:` line is for humans; the build dispatcher reads `metadata.testPlanFile`, `metadata.testPlanCases`, and `metadata.testPlanCoverage`. Phase 5 MUST write these via `bd update --metadata` for every bead, and apply the `kind:integration-test` label to the per-epic integration-test bead. Skipping this means the dispatcher's testFileLocks no-op, the validator's testArchitecture audit can't run, and integration-test beads pointlessly cycle through the builder stage.

## The Process

### Phase 1: Analyze Input Document

1. Read the entire document
2. Identify natural groupings (phases, features, components) → **epics**
3. Identify actionable items within each grouping → **issues**
4. Note explicit dependencies mentioned in text
5. Note technical details, acceptance criteria, edge cases

### Phase 2: Draft Structure (DO NOT EXECUTE YET)

Create a written draft:
```
Epic: [Name]
  - Issue: [Title]
    Dependencies: [list or "none - can start immediately"]
    Key details: [from source doc]
```

**Coverage matrix (REQUIRED):**
```
Source Section → Beads Location                | Est. Files | Est. ACs | Size
───────────────────────────────────────────────|────────────|──────────|──────
"Phase 1.1 Stripe Setup" → Epic 1, Issue 1.1  | 3          | 4        | ✓ OK
"Technical note: 30s timeout" → Issue 1.1 (AC) | —          | —        | —
"Phase 2 Full Pipeline"  → Epic 2, Issue 2.1  | 9          | 11       | ⚠️ SPLIT
```

**Every source section MUST appear.** Gaps = work not captured = failure.

**Detail density tracking (REQUIRED):**
```
Phase/Section     | Source Detail | Beads Detail | Status
──────────────────|───────────────|──────────────|────────
Phase 1 (4 items) | 450 words     | 380 words    | ✓ OK
Phase 2 (6 items) | 600 words     | 120 words    | ⚠️ SPARSE
Phase 3 (3 items) | 200 words     | 190 words    | ✓ OK
```

**Sparse phase = red flag.** If beads detail is <50% of source detail, the conversion lost information. Go back and capture missing details before proceeding.

**Test File Plan (REQUIRED):**

Rows are *test files*, not epics. Each row lists which epics extend it and which integration beads (if any) own cases in it.

```
Test file                              | Type        | Used by epics            | Integration beads
───────────────────────────────────────|─────────────|──────────────────────────|────────────────────
src/lib/stripe.test.ts                 | unit        | Epic 1, Epic 4           | —
src/services/webhook.test.ts           | unit        | Epic 2                   | —
src/features/auth/auth.integration.ts  | integration | Epic 4, Epic 7, Epic 12  | 4.5, 7.4, 12.3
```

Rules for filling this in:
- **Walk the codebase first.** Identify which existing test files cover the modules each epic will modify. Reuse them.
- **One row per existing or new test file**, never per epic. If three epics touch `src/lib/stripe.ts`, they share one row.
- **Integration files are domain-scoped.** Group epics that ship related user flows under one `<domain>.integration.test.<ext>`.
- **Every integration bead from Phase 2 must appear** in the "Integration beads" column of exactly one integration row.
- **Single-bead epics** still appear in the "Used by epics" column of whatever module file their tests extend — they just don't appear in any integration row. Note exemption explicitly.

**Per-bead structured Test Plan fields (REQUIRED):**

The `Test plan:` prose line in a bead description is for human readers. The build-dispatch orchestrator (which runs the testers/builders/validators) does NOT parse prose — it reads structured fields from each bead's bd metadata. For each bead, capture three fields alongside the prose line:

| Field | Type | Source | Example |
|-------|------|--------|---------|
| `testPlanFile` | string (path) | the file named in `Adds N cases to <file>` | `src/lib/stripe.test.ts` |
| `testPlanCases` | integer | the N | `3` |
| `testPlanCoverage` | string (semicolon-separated) | the `Covers: a; b; c` portion | `webhook signature validation; idempotent retry; failure logging` |

Plus a label for integration-test beads:

| Label | When | Why |
|-------|------|-----|
| `kind:integration-test` | the per-epic integration-test bead | Tells the dispatcher to skip the builder stage — these beads have no implementation, only end-to-end tests against already-merged sibling code |

These fields and labels are written in Phase 5 via `bd update --metadata` and `bd update --add-label`. Phase 2 is where they get *committed to* — the prose Test plan line and the structured fields must agree, or build-dispatch will dispatch testers at the wrong file and the validator's architecture audit will reject the bead.

### Phase 3: Subagent Review Passes (MANDATORY)

**DO NOT self-review.** Launch independent subagents for each pass using the Task tool. Self-review is biased; subagents catch what you miss.

**If Task tool unavailable:** Present full draft to user for manual review at each pass checkpoint. Do NOT substitute self-review for subagent review.

**Pass 1: Completeness + Detail Density Review** (launch subagent with Task tool)

Subagent mandate:
> "Review this beads structure against the source document. Verify: (1) every actionable item has a corresponding issue, (2) every technical note is captured, (3) every acceptance criterion is attached, (4) coverage matrix has no gaps, (5) **detail density check: for each phase, compare word count of source vs beads - flag any phase where beads is <50% of source as SPARSE**, (6) **sizing check: flag any issue with >5 files to modify, >6 acceptance criteria, >300 word description, or cross-module scope as OVERSIZED — these must be split before proceeding**, (7) **test architecture check: verify a Test File Plan exists with rows per test file (not per epic); every epic with ≥3 implementation beads has an integration-test bead that depends on all sibling implementation beads; beads modifying a module that already has a test file extend that file rather than introducing a new one; epics in the same feature domain share an integration test file rather than each owning their own; every integration bead appears in exactly one integration row of the plan; and single-bead-epic exemptions are explicitly noted**. Report any missing content, sparse phases, oversized issues, missing/misplaced integration-test beads, or unnecessary new test files."

**Detail density criteria for each issue:**
- Description must be substantive (>50 words) OR source item was genuinely simple
- Acceptance criteria count should roughly match source complexity
- Technical notes from source preserved verbatim, not summarized
- If source bullet has sub-bullets, issue must capture them all

Apply fixes from Pass 1 before proceeding.

**Pass 2: Dependencies Review** (launch subagent)

Subagent mandate:
> "Review dependencies in this beads structure. Identify: false sequential ordering (could be parallel), missing blockers, incorrect dependency direction, **backward phase dependencies (Phase N task depending on Phase N+M task)**, over-constrained chains. For each dependency, answer: Does A truly BLOCK B from starting?"

**Backward phase dependency = structural error.** If a Phase 1 task depends on a Phase 3 task, something is wrong:
- The phases are ordered incorrectly in the source document
- The task belongs in a different phase
- The dependency is incorrect

When backward phase dependencies are found, ask the user: "Should I update the source spec/design document to fix the phase ordering, or should we adjust the task placement?"

Apply fixes from Pass 2 before proceeding.

**Pass 3: Clarity + Density Review** (launch subagent)

Subagent mandate:
> "Review each issue for implementation readiness. Check: title clarity, description completeness, acceptance criteria verifiability. Flag: vague language, assumed knowledge, missing context. **Also flag any issue where description is <50 words but the corresponding source section had >100 words - this indicates lost detail.** **Sizing gate: verify every issue meets these limits — ≤5 files to modify, ≤6 acceptance criteria, ≤300 word description, single domain scope. Any issue exceeding these is OVERSIZED and must be split before execution.** **Test plan gate: verify every issue (including the integration-test bead) has a `Test plan:` line in the form `Adds N cases to <test-file>. Covers: <behaviors>.` The named file must appear in the Test File Plan: unit beads point at the module's existing test file, integration beads point at the feature domain's integration test file. Flag any issue missing the field, naming a file not listed in the plan, introducing a new file when the module/domain already has one, or with vague coverage like 'tests the feature'.** **Structured-field gate (build-dispatch contract): for every bead, verify the prose Test plan line yields concrete structured values — `testPlanFile` is a real path (no `<placeholder>`), `testPlanCases` is an integer, `testPlanCoverage` is a non-empty semicolon-separated list. The integration-test bead per epic must be flagged for the `kind:integration-test` label. Flag any bead where the structured fields can't be derived unambiguously from the prose — that bead will fail in Phase 5.** Could a fresh Sonnet agent execute each issue within 60% of its 200K token context window without referring back to the source document?"

Apply fixes from Pass 3 before proceeding.

**Optional Pass 4: Implementation Readiness** (for complex projects)

Subagent mandate:
> "For each issue: Could a fresh agent with zero codebase context pick this up and execute? Flag: missing file paths, assumed knowledge, vague verbs like 'update the thing'."

### Phase 4: User Checkpoint

**Before executing bd commands, present the final structure to the user:**

```
Ready to create:
- X epics
- Y issues
- Z blocking dependencies
- W items ready immediately (parallel start)

Coverage: All N source sections mapped ✓

Detail Density:
  Phase 1: 450 → 380 words (84%) ✓
  Phase 2: 600 → 550 words (92%) ✓
  Phase 3: 200 → 190 words (95%) ✓
  Overall: 1250 → 1120 words (90%) ✓

Sizing (Sonnet 60% context budget):
  All issues ≤5 files: ✓
  All issues ≤6 ACs: ✓
  All issues ≤300 words: ✓
  All issues single-domain: ✓
  Oversized issues split: 0 remaining

Test Architecture:
  Test File Plan complete (rows per file, not per epic): ✓
  Beads reuse existing module/domain test files: ✓
  Integration-test bead per multi-bead epic: ✓ (3/3 epics)
  Domain-shared integration files where applicable: ✓
  All beads declare Test plan field (prose): ✓
  All beads have derivable structured fields (testPlanFile/Cases/Coverage): ✓
  Integration-test beads tagged for kind:integration-test label: ✓ (3/3)

Proceed with creation? [User must confirm]
```

**If any phase shows <50% density, DO NOT proceed.** Go back and flesh out the sparse phases before asking for confirmation.

**If any issue is flagged OVERSIZED, DO NOT proceed.** Split it first using the decomposition strategies above.

**DO NOT skip this checkpoint.** User approval prevents creating wrong structure.

### Phase 5: Execute

Only after user confirmation:

1. **Create epics first:**
   ```bash
   bd create "Epic title" -t epic -p <priority> -d "Description"
   ```

2. **Create issues with parent-child links AND structured Test Plan metadata:**
   ```bash
   bd create "Issue title" -t task -d "Description (with Test plan: line)" \
     --acceptance "- [ ] Criterion 1"
   # Capture the new issue id, then write the structured fields the build dispatcher reads.
   # Use the @file pattern — testPlanCoverage often contains commas/semicolons that break inline JSON.
   cat > /tmp/bd-meta-{issue-id}.json << 'ENDJSON'
   {
     "testPlanFile": "src/lib/stripe.test.ts",
     "testPlanCases": 3,
     "testPlanCoverage": "webhook signature validation; idempotent retry; failure logging"
   }
   ENDJSON
   bd update {issue-id} --metadata @/tmp/bd-meta-{issue-id}.json
   rm /tmp/bd-meta-{issue-id}.json
   bd dep add <epic-id> <issue-id> --type parent-child
   ```

   **For integration-test beads, also add the kind label:**
   ```bash
   bd update {integration-bead-id} --add-label "kind:integration-test"
   ```
   This label tells build-dispatch to skip the builder stage for that bead — the tester writes passing tests against already-merged sibling code, then the validator runs.

3. **Add blocker dependencies** (only TRUE blockers):
   ```bash
   bd dep add <prerequisite-id> <dependent-id>
   ```

4. **Verify:**
   ```bash
   bd dep cycles                  # Must return empty
   bd ready --json                # Check expected items ready
   bd show <bead-id> --json       # Spot-check 2-3 beads — confirm metadata.testPlanFile
                                  # is populated and prose Test plan line agrees with it
   bd list --label "kind:integration-test" --json   # Should list exactly the integration-test beads
   ```

### Phase 6: Generate Report

**A. Creation Summary**
```
Created: X epics, Y issues
  Epic 1: [Name] (N issues)
  ...
```

**B. Dependency Graph** (show blocking relationships and parallel opportunities)

**C. Ready Work Queue** (items with no blockers)

**D. Coverage Verification**
```
Source sections: N
Mapped to beads: N ✓
Information loss: None
```

**E. Build-Dispatch Contract Verification**
```
Beads with metadata.testPlanFile populated: N/N ✓
Integration-test beads labeled kind:integration-test: M/M ✓
Spot-check (bd show on 2-3 random beads): metadata + prose Test plan agree ✓
```
If any of these is not 100%, the build-dispatch run will fail at dispatch time (testers will guess at file placement, integration-test beads will run a useless builder stage). Fix before handing off to /build.

## Loophole Closures

**"I'll review it myself to save time"** → WRONG. Self-review is biased. Launch subagents.

**"The review passes are just a formality"** → WRONG. Apply fixes between each pass. Document what changed.

**"User checkpoint slows things down"** → WRONG. Wrong structure wastes more time. Get confirmation.

**"These items obviously need to be sequential"** → WRONG. Prove it. State why A must complete before B can START.

**"I captured the key points"** → WRONG. Capture ALL points. Lossless means lossless.

**"This is simple enough for one pass"** → WRONG. Even simple docs need review for dependencies.

**"This bead is big but the builder will figure it out"** → WRONG. Sonnet has a fixed 200K context window. A bead touching 8 files with 10 ACs WILL exhaust the builder's budget. Split it.

**"Splitting creates too many small beads"** → WRONG. 10 focused beads that each complete successfully > 5 oversized beads where 3 fail from context exhaustion. More beads also means more parallelism.

**"Phase 1 legitimately depends on Phase 3"** → WRONG. Backward phase dependencies signal incorrect ordering. Ask user if source doc needs updating.

**"This phase is simpler, it doesn't need much detail"** → WRONG. If the source has detail, capture it. A 600-word source section becoming a 120-word beads issue = 80% information loss. Sparse issues cause implementation failures.

**"I'll add more detail when implementing"** → WRONG. The beads issue IS the specification. If detail isn't in the issue, it's not in the spec. Capture it now or lose it.

**"Each bead should have its own test file for isolation"** → WRONG. Tests follow the codebase, not the bead — a bead modifying `src/lib/stripe.ts` extends `src/lib/stripe.test.ts`, even if four other beads from three other epics also extend it. Isolation comes from `describe`/`context` blocks, not separate files.

**"Each epic should own its own integration test file"** → WRONG at scale. At 100 epics that's 100 integration files. Group epics by feature domain (auth, courts, billing) and let them share `<domain>.integration.test.<ext>` — the integration *bead* stays per-epic so failures point at a specific outcome, but the *cases* live together.

**"Unit tests on each bead are enough"** → WRONG. Green unit tests + broken integration is the default failure mode. The integration-test bead is non-negotiable for multi-bead epics — it's the only artifact that proves the epic works as a whole.

**"The builder will figure out where tests go"** → WRONG. Without a Test plan field naming the file and cases, builders default to creating a new file. Make the target explicit in every bead.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Flat issue list | Use epics with parent-child |
| Self-review | Launch subagents |
| Sequential-by-default | Prove each blocker |
| Backward phase dependencies | Reorder phases or move task; offer to update source doc |
| Summarizing details | Preserve exact wording |
| Sparse descriptions (<50% density) | Compare word counts; flesh out before proceeding |
| Oversized beads (>5 files, >6 ACs) | Split along natural seams; use decomposition strategies |
| Full-stack beads (backend + frontend) | Split into backend bead + frontend bead with dependency |
| Skipping user checkpoint | Always get confirmation |
| Rushing for time pressure | Quality over speed |
| New test file per bead/epic | Unit tests follow the module; integration tests follow the feature domain — beads extend, not create |
| Per-epic integration files | Group epics by feature domain into one shared `<domain>.integration.test.<ext>` |
| No integration-test bead | Add one per multi-bead epic, depending on all siblings |
| Missing Test plan field | Every bead names its target test file (already in the plan) and cases upfront |
| Prose Test plan but no metadata | Phase 5 also writes `metadata.testPlanFile/Cases/Coverage` and `kind:integration-test` label so build-dispatch can read them |

## Quick Reference

```
1. READ entire source document
2. DRAFT structure with coverage matrix
3. SUBAGENT Pass 1: Completeness → Apply fixes
4. SUBAGENT Pass 2: Dependencies → Apply fixes
5. SUBAGENT Pass 3: Clarity → Apply fixes
6. (Optional) SUBAGENT Pass 4: Implementation readiness
7. USER CHECKPOINT: Present structure, get confirmation
8. EXECUTE bd commands (epics → issues + structured Test Plan metadata + kind:integration-test labels → deps)
9. VERIFY no cycles, expected items ready, every bead has metadata.testPlanFile
10. REPORT summary, graph, queue, coverage, build-dispatch contract
```

**Remember:** Lossless. Right-sized (≤5 files, ≤6 ACs). Module-owned unit tests, domain-owned integration tests. Integration-test bead per multi-bead epic. Test plan on every bead — both prose and structured metadata. Subagent reviews. User checkpoint. Maximum parallelization.