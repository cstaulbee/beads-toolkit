# Phase Detail — Build Dispatch Orchestrator

This file contains the detailed pseudocode for each protocol phase. The orchestrator reads this when executing a build.

## Table of Contents

- [Phase 0: Init](#phase-0-init)
- [Phase 1a: Build Epic's Beads](#phase-1a-build-epics-beads)
- [Phase 1b: Quality Gate](#phase-1b-quality-gate)
- [Phase 1c: PR to Main](#phase-1c-pr-to-main)
- [Phase 1d: Auto-Merge & Cleanup](#phase-1d-auto-merge--cleanup)
- [Phase 2: Summary](#phase-2-summary)
- [Helper Functions](#helper-functions)

---

## Phase 0: Init

**Entry**: User invokes `/build` — beads and epics MUST already exist
**Exit**: Build epic created, epic-ordered plan determined

Beads are created by design-to-beads or spec-to-beads *before* this skill runs. If no open beads exist, abort: "No open beads found. Run design-to-beads or spec-to-beads first."

Steps:

1. Create build epic:
   ```bash
   bd create --title="Build: {feature-name}" --type=epic
   bd update {build-epic} --metadata @/tmp/bd-meta-build.json
   # See state-schemas.md for the build epic metadata shape
   ```

2. Parse input:
   - No args or `--resume` → resume mode (use existing beads)
   - `--beads ForgeFlow-1,ForgeFlow-2` → specific mode (only listed beads)

3. Run `bd stats` and `bd list --status=open --json` to assess backlog.

4. Collect and order:
   ```
   allBeads = bd list --status=open --json
   Detect dependency cycles → abort if found
   epicGroups = groupBeadsByEpic(allBeads)  // beads with no parent → implicit "ungrouped" epic
   epicOrder = topologicalSort(epicGroups, crossEpicDependencies)
   // Epic A before Epic B if ANY bead in B depends on ANY bead in A
   ```

5. Print plan (informational, no confirmation needed):
   ```
   Build plan (epic milestones):
     Epic 1: {title} ({N} beads, no dependencies)
     Epic 2: {title} ({N} beads, depends on Epic 1)
     Mode: TDD-first | Concurrency: {maxConcurrency}
   ```

6. Store epicOrder + config in build epic metadata. Proceed to Phase 1 immediately.

---

## Phase 1: Epic Loop

```
update build epic: phase = "epic-loop"

for epicIndex, epicId in enumerate(epicOrder):
  update build epic: currentEpicIndex = epicIndex
  epicBeads = epicGroups[epicId]
  Print: "[EPIC {epicIndex+1}/{len(epicOrder)}] Starting: {epicTitle} ({len(epicBeads)} beads)"

  Phase 1a → Phase 1b → Phase 1c → Phase 1d

  epicResults[epicId] = {prUrl, beadsCompleted, beadsFailed, gatesPassed}
  update build epic: epicResults

  // Cleanup this epic's worktrees (merger handles it)
  Print: "[GATE] Epic {epicIndex+1} complete → next epic"

goto Phase 2
```

---

## Phase 1a: Build Epic's Beads

**Entry**: Epic selected, epicBeads populated
**Exit**: All beads merged or failed, or abort threshold hit

### Init Epic Build Branch

The branch must exist before any beads dispatch, because dependent beads branch their worktrees from it.

```
epicBuildBranch = "build/{feature_name}/{epicShortName}"

Agent(subagent_type="beads-merger", run_in_background=true,
  prompt="type: init-epic
          epic_name: {epicShortName}
          base_branch: {baseBranch}  // 'main' for first epic, updated main for subsequent
          feature_name: {feature_name}
          Create a new build branch for this epic.")
Block wait for MERGE_COMPLETE.
```

### Poll Loop

```
activePipelines = {}
testFileLocks = Set()    // testPlanFile paths held by in-flight testers
mergeQueue = []
mergeInFlight = null
mergedSet = Set()
failedCount = 0

while true:
  // 1. Fill slots with eligible beads
  eligible = eligibleBeads(epicBeads, mergedSet, mergeQueue, mergeInFlight, activePipelines, testFileLocks)
  freeSlots = maxConcurrency - len(activePipelines)
  for bead in eligible[:freeSlots]:
    pipeline = dispatchBeadPipeline(bead)
    if pipeline is null:
      // Config error (e.g., missing testPlanFile); count as failed and continue
      failedCount++
      continue
    activePipelines[bead.id] = pipeline

  // 2. Poll all active pipelines (batch all TaskOutput calls in one response)
  for pipeline in activePipelines:
    TaskOutput(task_id=pipeline.taskId, block=false, timeout=5000)
    if completion marker found:
      advancePipeline(pipeline, result)  // releases testFileLocks if leaving testing stage
    if pipeline.stage == "validated":
      mergeQueue.append(pipeline.beadId)
      remove from activePipelines
    if pipeline.stage == "failed":
      failedCount++
      // Release any test-file locks the pipeline still held (failure during testing)
      for f in pipeline.testTargetFiles: testFileLocks.remove(f)
      bugId = bd create --title="Fix: {bead title}" --type=bug --priority=1
      // Inherit testPlanFile so remediation tester targets the same file
      bd update {bugId} --metadata @/tmp/bd-meta-{bugId}.json  // {testPlanFile, testPlanCases, testPlanCoverage}
      epicBeads.append({id: bugId, dependencies: [], metadata: {testPlanFile: ...}})
      remove from activePipelines
    if stage timeout exceeded:
      TaskStop(task_id=pipeline.taskId)
      for f in pipeline.testTargetFiles: testFileLocks.remove(f)
      bd create --title="Timeout: {bead title}" --type=bug --priority=1
      failedCount++
      remove from activePipelines

  // 3. Process merge queue (serialized — one at a time)
  if mergeInFlight is not null:
    TaskOutput(task_id=mergeInFlight.taskId, block=false, timeout=5000)
    if MERGE_COMPLETE found:
      if success:
        mergedSet.add(mergeInFlight.beadId)
        bd update {mergeInFlight.beadId} --add-label "merged"
        Print: "[MERGE] Bead {id} merged into {epicBuildBranch}"
      else:
        failedCount++
        bd create --title="Merge fix: {bead title}" --type=bug --priority=1
        epicBeads.append(...)
      mergeInFlight = null

  if mergeInFlight is null and len(mergeQueue) > 0:
    nextBeadId = mergeQueue.shift()
    taskId = Agent(subagent_type="beads-merger", run_in_background=true,
      prompt="type: merge-bead
              Merge bead/{nextBeadId} into '{epicBuildBranch}'.
              Run typecheck after merge.
              Bead branch: bead/{nextBeadId}
              Build branch: {epicBuildBranch}")
    mergeInFlight = {beadId: nextBeadId, taskId}

  // 4. Abort threshold (after ≥ half resolved)
  totalResolved = len(mergedSet) + failedCount
  if totalResolved >= ceil(len(epicBeads) / 2):
    if failedCount / totalResolved > abortThreshold / 100:
      Print: "[GATE] Phase 1a → ABORT: threshold exceeded"
      break

  // 5. Exit condition
  if len(activePipelines) == 0 and mergeInFlight is null
     and len(mergeQueue) == 0 and len(eligible) == 0:
    break

  sleep(15 seconds)

Print: "[GATE] Phase 1a complete: {len(mergedSet)} merged, {failedCount} failed"
```

### Pre-Quality-Gate Audit

Before entering Phase 1b, verify every bead in mergedSet has both `validation:PASS` and `merged` labels:

```
for beadId in mergedSet:
  beadData = bd show {beadId} --json
  verify "validation:PASS" in labels AND "merged" in labels
  if missing → log warning, investigate

if len(mergedSet) == 0:
  Print: "[GATE] Phase 1a → 1b: FAIL — no merged beads"
  skip to next epic
```

---

## Phase 1b: Quality Gate

**Entry**: Pre-audit passed, all beads merged into epic build branch
**Exit**: Full quality gate passes, or remediation exhausted

```
Agent(subagent_type="beads-merger", run_in_background=true,
  prompt="type: quality-gate
          Run full quality gate (typecheck + lint + test) on '{epicBuildBranch}'.
          IMPORTANT: Use explicit Bash timeouts: npm install=600000, typecheck/lint/test=300000.
          Build branch: {epicBuildBranch}")
Block wait for MERGE_COMPLETE.

if quality_passed:
  Print: "[GATE] Phase 1b: PASS"
  goto Phase 1c

// Remediation loop (up to maxRemediationCycles)
for cycle in 1..maxRemediationCycles:
  Print: "[GATE] Phase 1b: FAIL — remediation cycle {cycle}"
  failures = mergeResult.details.failures

  // Create bug beads for each failure group
  remBeads = []
  for group in groupByFile(failures):
    bugId = bd create --title="Gate fix: {summary}" --type=bug --priority=1
    remBeads.append(bugId)

  // Run through TDD pipeline + incremental merge
  for remBead in remBeads:
    dispatchBeadPipeline(remBead)
    poll until validated or failed

  // Merge successful remediation beads
  for remBead in validated(remBeads):
    Agent(subagent_type="beads-merger", ..., prompt="type: merge-bead ...")
    Block wait.

  // Re-run full quality gate
  Agent(subagent_type="beads-merger", ..., prompt="type: quality-gate ...")
  Block wait.

  if quality_passed:
    Print: "[GATE] Phase 1b: PASS (after {cycle} remediation cycles)"
    goto Phase 1c

// Exhausted → escalate
AskUserQuestion: "Quality gate still failing after {max} cycles. Failures: {failures}. How to proceed?"
```

---

## Phase 1c: PR to Main

```
Agent(subagent_type="beads-merger", run_in_background=true,
  prompt="type: create-pr
          Create PR from '{epicBuildBranch}' to main.
          Feature: {epicTitle}
          Summary: {len(mergedSet)} completed, {failedCount} failed, gate: PASS
          Include bead list in PR body.")
Block wait for MERGE_COMPLETE.
prUrl = result.details.pr_url

Print: "[GATE] Phase 1c → PR created: {prUrl}"
```

---

## Phase 1d: Auto-Merge & Cleanup

```
Agent(subagent_type="beads-merger", run_in_background=true,
  prompt="type: merge-pr
          Merge PR {prUrl} into main using gh pr merge --merge.
          After merge, pull main to update local.
          Clean up ALL worktrees and bead branches for this epic.
          Epic build branch: {epicBuildBranch}
          Bead branches: {list}
          Worktrees: {list}
          Delete epic build branch after merge.
          Emit MERGE_COMPLETE with type: merge-pr.")
Block wait for MERGE_COMPLETE.

if failed:
  Print: "[GATE] Phase 1d: PR merge failed"
  goto Phase 2 with failure noted

Print: "[GATE] Phase 1d: PR merged + cleanup complete"
// Next epic branches from updated main
```

---

## Phase 2: Summary

```
update build epic: phase = "summary"
bd close {build-epic} --reason="Build complete"
bd sync

Print:
  Build Complete!
    Feature: {feature-name}
    Epics: {completed}/{total}

    Epic Results:
      Epic 1: {title} — PR {url} ({N} beads, gate: PASS)
      ...

    Total beads: {completed}/{total}
    Failed: {count}
    All PRs auto-merged and branches cleaned up.
```

---

## Helper Functions

### eligibleBeads

```
eligibleBeads(epicBeads, mergedSet, mergeQueue, mergeInFlight, activePipelines, testFileLocks):
  return [b for b in epicBeads
          if b.status == "open"
          and b.id not in activePipelines
          and b.id not in mergeQueue
          and b.id not in mergedSet
          and (mergeInFlight is null or b.id != mergeInFlight.beadId)
          and all(dep in mergedSet for dep in b.dependencies)
          and (b.metadata.testPlanFile is null
               or b.metadata.testPlanFile not in testFileLocks)]
```

The `testFileLocks` clause keeps two beads with the same `testPlanFile` from racing on the shared file. The lock is released when the holder leaves the testing stage.

### dispatchBeadPipeline

```
isIntegrationTest = "kind:integration-test" in bead.labels
testTargetFiles = []
if bead.metadata.testPlanFile:
  testTargetFiles = [bead.metadata.testPlanFile]
elif tddEnabled:
  // Missing testPlanFile under TDD = config error. Don't dispatch a tester
  // that will guess at file placement and recreate the singleton problem.
  Print: "[ERROR] Bead {bead.id} has no metadata.testPlanFile under TDD; aborting bead."
  bd update {bead.id} --add-label "config-error"
  return null

if tddEnabled:
  bd update {bead.id} --status=in_progress --add-label "stage:testing"
  bd update {bead.id} --metadata @/tmp/bd-meta-{id}.json
  // metadata: {"worktree":"../forgeflow-{id}","branch":"bead/{id}","epicBuildBranch":"..."}
  // Preserve testLevels/testPriority/testPlanFile/testPlanCases/testPlanCoverage from design-to-beads.

  // Acquire test-file lock for the duration of the testing stage
  for f in testTargetFiles: testFileLocks.add(f)

  if isIntegrationTest:
    testerPrompt = "Test Agent (INTEGRATION-TEST) for bead {beadId}. Worktree: {worktreePath}.
                    Run `bd show {beadId} --json` for spec and metadata.
                    Sibling implementation is already merged into the epic build branch you branched from.
                    Extend metadata.testPlanFile with end-to-end tests covering metadata.testPlanCoverage.
                    Tests should PASS against the current code (this is integration verification, not red TDD).
                    Emit TEST_COMPLETE marker."
  else:
    testerPrompt = "Test Agent for bead {beadId}. Worktree: {worktreePath}.
                    Run `bd show {beadId} --json` for spec and acceptance criteria.
                    metadata.testLevels: which test levels to generate.
                    metadata.testPriority: depth (P0=full, P3=minimal).
                    metadata.testPlanFile: the EXACT file to extend. Do NOT create a new file unless this
                      file genuinely does not yet exist. Per-bead singleton test files are forbidden.
                    metadata.testPlanCases / metadata.testPlanCoverage: target case count and behaviors.
                    Write FAILING tests for every AC. Emit TEST_COMPLETE marker."

  taskId = Agent(subagent_type="beads-tester", run_in_background=true, prompt=testerPrompt)
  return {beadId, stage="testing", taskId, retryCount=0, stageStartTime=now(),
          testTargetFiles, isIntegrationTest}

else:
  // Non-TDD: skip to Builder. testFileLocks not used in this mode.
  bd update {bead.id} --status=in_progress --add-label "stage:building"
  bd update {bead.id} --metadata @/tmp/bd-meta-{id}.json
  taskId = Agent(subagent_type="beads-builder", run_in_background=true,
    prompt="Builder for bead {beadId}. Worktree: {worktreePath}.
            Run `bd show {beadId} --json`. Implement until done. Emit BUILD_COMPLETE.")
  return {beadId, stage="building", taskId, retryCount=0, stageStartTime=now(),
          testTargetFiles=[], isIntegrationTest=false}
```

### advancePipeline

```
switch pipeline.currentStage:

  case "testing":
    testData = parseTestComplete(result)
    // Write to bd (use @file — criteriaMapping has user-generated strings)
    write testData to /tmp/bd-meta-{id}.json
    bd update {id} --metadata @/tmp/bd-meta-{id}.json
    rm /tmp/bd-meta-{id}.json

    // Verify metadata was written before advancing
    bd show {id} --json → confirm metadata.testing.status == "completed"
    // If missing → something went wrong. Do not dispatch builder.

    // Release test-file locks BEFORE dispatching the next stage.
    // Sibling beads with the same testPlanFile become eligible the moment this completes.
    for f in pipeline.testTargetFiles: testFileLocks.remove(f)

    // Integration-test beads skip the builder. Their tester wrote PASSING tests against
    // already-merged sibling code. Dispatch validator directly.
    if pipeline.isIntegrationTest:
      bd update {id} --remove-label "stage:testing" --add-label "stage:validating"
      bd update {id} --metadata '{"building":{"status":"closed","commitSha":"(no-impl: integration-test)","discoveredBeads":[]}}'
      taskId = Agent(subagent_type="beads-validator",
        prompt="Validator for bead {beadId} (INTEGRATION-TEST). Worktree: {worktreePath}.
                No implementation — sibling impl is already merged. Re-run the tests.
                Audit: did all tests land in metadata.testPlanFile?
                Were any singleton test files created?
                Do tests cover metadata.testPlanCoverage end-to-end?
                Emit VALIDATE_COMPLETE marker including testArchitecture: PASS|VIOLATION.")
      pipeline.stage = "validating"
      pipeline.taskId = taskId
      pipeline.stageStartTime = now()
      break

    bd update {id} --remove-label "stage:testing" --add-label "stage:building"
    taskId = Agent(subagent_type="beads-builder", run_in_background=true,
      prompt="You are the Builder for bead {beadId}. Worktree: {worktreePath}.
              Run `bd show {beadId} --json` for spec + testing metadata.
              Make all tests pass. Do NOT modify test files. Emit BUILD_COMPLETE.")
    pipeline.stage = "building"
    pipeline.taskId = taskId
    pipeline.stageStartTime = now()

  case "building":
    buildData = parseBuildComplete(result)
    bd update {id} --metadata '{"building":{"status":"closed","commitSha":"...","discoveredBeads":[]}}'

    bd update {id} --remove-label "stage:building" --add-label "stage:validating"
    taskId = Agent(subagent_type="beads-validator", run_in_background=true,
      prompt="You are the Validator for bead {beadId}. Worktree: {worktreePath}.
              Run `bd show {beadId} --json` for spec + all metadata.
              Audit completeness and test quality.
              TEST ARCHITECTURE AUDIT (testArchitecture: PASS|VIOLATION):
                metadata.testPlanFile names where the bead's tests must live.
                Verify all of this bead's tests landed in that file.
                No new singleton test files (one-per-bead) may be introduced.
                If any of these fail, emit VIOLATION with details in testArchFindings.
              Emit VALIDATE_COMPLETE marker including testArchitecture: PASS|VIOLATION.")
    pipeline.stage = "validating"
    pipeline.taskId = taskId

  case "validating":
    valData = parseValidateComplete(result)
    write valData to /tmp/bd-meta-{id}.json  // @file — findings has free-text
    bd update {id} --metadata @/tmp/bd-meta-{id}.json
    rm /tmp/bd-meta-{id}.json
    routeValidation(pipeline, valData)
```

### routeValidation

```
// Test architecture violations always route back to the tester, regardless of severity.
// Per-bead singleton test files defeat the entire Test File Plan; never let one through.
if valData.testArchitecture == "VIOLATION" and pipeline.retryCount < 1:
  pipeline.retryCount++
  bd update {id} --metadata '{"retry_count": 1}'
  bd update {id} --remove-label "stage:validating" --add-label "stage:testing"
  // Re-acquire test-file lock for retry tester
  for f in pipeline.testTargetFiles: testFileLocks.add(f)
  Agent(subagent_type="beads-tester", run_in_background=true,
    prompt="Test Agent for bead {beadId} (RETRY — test architecture violation).
            Worktree: {worktreePath}. Validator found: {valData.testArchFindings}.
            Move ALL of this bead's tests into metadata.testPlanFile. Delete any singleton
            test files this bead created. Emit TEST_COMPLETE.")
  pipeline.stage = "testing"
  return

if valData.severity == "PASS":
  bd update {id} --remove-label "stage:validating" --add-label "validation:PASS"
  bd show {id} --json  // verify label was written
  pipeline.stage = "validated"

else if valData.severity == "MINOR_ISSUES" and pipeline.retryCount < 1:
  pipeline.retryCount++
  bd update {id} --metadata '{"retry_count": 1}'

  if valData.testQuality == "GAPS_FOUND":
    // Test gaps → retry from Tester
    bd update {id} --remove-label "stage:validating" --add-label "stage:testing"
    for f in pipeline.testTargetFiles: testFileLocks.add(f)
    Agent(subagent_type="beads-tester", run_in_background=true,
      prompt="Test Agent for bead {beadId} (RETRY). Worktree: {worktreePath}.
              Validator found test gaps. Write improved tests INTO metadata.testPlanFile
              (do not introduce new singleton files). Emit TEST_COMPLETE.")
    pipeline.stage = "testing"
  else:
    // Impl issue → retry from Builder
    bd update {id} --remove-label "stage:validating" --add-label "stage:building"
    Agent(subagent_type="beads-builder", run_in_background=true,
      prompt="Builder for bead {beadId} (RETRY). Worktree: {worktreePath}.
              Validator findings: {valData.findings}. Fix issues, pass tests.
              Do NOT modify test files. Emit BUILD_COMPLETE.")
    pipeline.stage = "building"

else:
  // MAJOR_ISSUES or retry exhausted
  bugId = bd create --title="Fix: {bead title}" --type=bug --priority=1 --description="{findings}"
  bd update {id} --add-label "superseded-by:{bugId}"
  bd close {id} --reason="Superseded by {bugId} after validation failure"
  pipeline.stage = "failed"
```
