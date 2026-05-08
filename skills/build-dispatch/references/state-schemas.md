# State Schemas — Build Dispatch Orchestrator

## Build Epic Metadata

Stored on the build epic via `bd update {id} --metadata @/tmp/bd-meta-build.json`.

```json
{
  "phase": "init|epic-loop|summary",
  "feature_name": "...",
  "epicOrder": ["epic-id-1", "epic-id-2"],
  "currentEpicIndex": 0,
  "epicResults": {},
  "remediationCycle": 0,
  "maxRemediationCycles": 3,
  "tddEnabled": true,
  "abortThreshold": 50,
  "maxConcurrency": 3,
  "stageTimeout": 600000,
  "last_updated": "ISO timestamp"
}
```

## Per-Bead Metadata

Each agent writes its stage output here. The next agent reads it on startup via `bd show {id} --json`.

```json
{
  "worktree": "../forgeflow-{bead-id}",
  "branch": "bead/{bead-id}",
  "epicBuildBranch": "build/{feature}/{epic}",
  "testLevels": ["unit-backend", "integration-api"],
  "testPriority": "P1",
  "testPlanFile": "src/lib/stripe.test.ts",
  "testPlanCases": 3,
  "testPlanCoverage": "webhook signature validation; idempotent retry; failure logging",
  "testing": {
    "status": "completed|failed",
    "testFiles": ["path1", "path2"],
    "testLevels": ["unit-backend", "integration-api"],
    "testDepth": "P1",
    "criteriaMapping": {"AC-1": "test name"},
    "unmappedCriteria": []
  },
  "building": {
    "status": "closed|failed",
    "commitSha": "a1b2c3d",
    "discoveredBeads": []
  },
  "validating": {
    "severity": "PASS|MINOR_ISSUES|MAJOR_ISSUES",
    "testQuality": "PASS|GAPS_FOUND",
    "testArchitecture": "PASS|VIOLATION",
    "testArchFindings": "e.g., 'created src/foo/bar.test.ts instead of extending src/foo.test.ts'",
    "findings": "summary of issues"
  },
  "retry_count": 0
}
```

The `testPlanFile`, `testPlanCases`, and `testPlanCoverage` fields are populated by design-to-beads when it creates the bead. The orchestrator reads them at dispatch time to:
- Tell the tester which file to extend (no per-bead singleton files)
- Acquire/release a `testFileLocks` entry that serializes testers across beads sharing the file
- Audit compliance via `validating.testArchitecture`

Beads with the `kind:integration-test` label skip the builder stage entirely — the tester writes passing tests against already-merged sibling code, and the validator runs immediately after.

## Writing Metadata Safely (Windows)

On Windows Git Bash, inline JSON in `bd update --metadata '...'` breaks when values contain parentheses, slashes, exclamation marks, or other shell metacharacters. The fix is simple: always write JSON to a temp file first.

**Simple values (machine-generated, no special chars)** — inline is OK:
```bash
bd update {id} --metadata '{"building":{"status":"closed","commitSha":"a1b2c3d"}}'
```

**Complex values (user-generated strings, AC descriptions, validator findings)** — use `@file`:
```bash
cat > /tmp/bd-meta-{beadId}.json << 'ENDJSON'
{
  "testing": {
    "status": "completed",
    "testFiles": ["src/modules/ai/migration.test.ts"],
    "criteriaMapping": {
      "AC-1": "barrel files exist with re-exports",
      "AC-2": "service re-exports from all 6 migrated services"
    },
    "unmappedCriteria": []
  }
}
ENDJSON
bd update {id} --metadata @/tmp/bd-meta-{beadId}.json
rm /tmp/bd-meta-{beadId}.json
```

**Rule of thumb**: If the JSON contains any values from agent output (criteriaMapping, findings, bead titles), use `@file`. If all values are machine-generated (status enums, SHAs, file paths with no spaces), inline is fine.

## Per-Bead BD Operations Reference

| Operation | When | Notes |
|-----------|------|-------|
| `bd update {id} --status=in_progress --add-label "stage:testing"` | Tester dispatch | |
| `bd update {id} --metadata @/tmp/bd-meta-{id}.json` | Tester dispatch | worktree, branch, epicBuildBranch |
| `bd update {id} --remove-label "stage:testing" --add-label "stage:building"` | TEST_COMPLETE | |
| `bd update {id} --metadata @/tmp/bd-meta-{id}.json` | TEST_COMPLETE | Use @file (criteriaMapping) |
| `bd update {id} --remove-label "stage:building" --add-label "stage:validating"` | BUILD_COMPLETE | |
| `bd update {id} --metadata '{"building":{...}}'` | BUILD_COMPLETE | Inline OK (machine values) |
| `bd update {id} --remove-label "stage:validating" --add-label "validation:PASS"` | VALIDATE_COMPLETE(PASS) | |
| `bd update {id} --metadata @/tmp/bd-meta-{id}.json` | VALIDATE_COMPLETE | Use @file (findings) |
| `bd update {id} --add-label "merged"` | MERGE_COMPLETE (per-bead) | |

## Completion Marker Schemas

### TEST_COMPLETE
```
<!-- TEST_COMPLETE:{"beadId":"...","status":"completed|failed","testFiles":[...],"testLevels":[...],"testDepth":"P0-P3","criteriaMapping":{...},"unmappedCriteria":[...]} -->
```

### BUILD_COMPLETE
```
<!-- BUILD_COMPLETE:{"beadId":"...","status":"closed|failed","commitSha":"...","discoveredBeads":[...]} -->
```

### VALIDATE_COMPLETE
```
<!-- VALIDATE_COMPLETE:{"beadId":"...","severity":"PASS|MINOR_ISSUES|MAJOR_ISSUES","testQuality":"PASS|GAPS_FOUND","testArchitecture":"PASS|VIOLATION","testArchFindings":"..."} -->
```

`testArchFindings` is required when `testArchitecture == "VIOLATION"` and may be omitted otherwise. The orchestrator routes any VIOLATION back to the tester with the findings, regardless of severity.

### MERGE_COMPLETE
```
<!-- MERGE_COMPLETE:{"type":"init-epic|merge-bead|quality-gate|create-pr|merge-pr|cleanup","status":"SUCCESS|FAILED","details":{...}} -->
```
