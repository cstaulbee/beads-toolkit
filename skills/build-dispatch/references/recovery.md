# Compaction Recovery — Build Dispatch Orchestrator

When you detect you've lost context mid-build (conversation seems to start mid-build, or you can't recall the current state), reconstruct everything from bd. Every important state transition is persisted via labels and metadata, making full recovery possible.

## Recovery Protocol

### Step 1: Read Build Epic

```bash
bd show {build-epic} --json
```

Extract: phase, feature_name, epicOrder, currentEpicIndex, config (tddEnabled, abortThreshold, maxConcurrency, stageTimeout).

If the build epic is unknown, search for it:
```bash
bd list --type=epic --status=open --json | grep "Build:"
```

### Step 2: Query Bead States

Run these in parallel:
```bash
bd list --status=in_progress --json    # beads currently being worked on
bd list --label "validation:PASS" --json   # merge-ready beads
bd list --label "merged" --json            # already-merged beads
```

### Step 3: Verify Metadata Integrity

For each bead with `validation:PASS` label, confirm the metadata actually backs it up:

```bash
bd show {id} --json → check metadata.validating exists
```

If `metadata.validating` is missing, the label is stale (the validation never actually completed — it may have been set prematurely or corrupted). Fix it:
```bash
bd update {id} --remove-label "validation:PASS" --add-label "stage:validating"
# Queue for re-validation
```

### Step 4: Derive Pipeline States

For each in-progress bead, determine its actual state from labels + metadata:

| Labels present | Metadata keys | State | Action |
|---------------|---------------|-------|--------|
| `merged` + `validation:PASS` | validating exists | Done | Add to mergedSet |
| `validation:PASS` (no `merged`) | validating exists | Validated | Add to mergeQueue |
| `validation:PASS` (no `merged`) | validating missing | Stale label | Remove label, re-validate |
| `stage:validating` | building exists | Validating | Re-dispatch validator |
| `stage:building` | testing exists | Building | Re-dispatch builder |
| `stage:testing` | — | Testing | Re-dispatch tester |
| No stage label, status open | — | Undispatched | Dispatch when eligible |

### Step 5: Rebuild In-Memory State

```
activePipelines = {beads in testing/building/validating stages}
mergeQueue = {beads with validation:PASS but not merged}
mergeInFlight = null  // safe to assume nothing in-flight after crash
mergedSet = {beads with both merged + validation:PASS labels}
failedCount = count of failed/closed beads in this epic
```

### Step 6: Print Recovery Audit

```
Recovery audit:
  Build epic: {id}, phase: {phase}
  Current epic: {currentEpicIndex+1}/{len(epicOrder)}
  Beads verified validated: {N}
  Beads re-queued for validation: {N}
  Beads in active pipeline: {N}
  Beads undispatched: {N}
  Beads merged: {N}
  Failed: {N}
```

### Step 7: Resume

Resume at the current phase and epic index. The poll loop will pick up from the rebuilt state.

## Why This Works

Every state transition writes to bd before proceeding:
- Dispatch → status=in_progress + stage label + metadata
- Stage advance → label swap + metadata write
- Validation pass → validation:PASS label + validating metadata
- Merge complete → merged label

Because writes happen before the next dispatch, and labels are authoritative, the worst case after a crash is re-running a stage that already completed (which is safe — agents are idempotent in their worktrees).
