---
name: epic-quality-pass
description: Audits a beads epic for build readiness — scores each child bead on a 0–100 confidence rubric (sizing, spec concreteness, context completeness, risk), proposes specific remediations to elevate every bead to ≥95, and verifies the epic's beads collectively implement the source plan (coverage, traceability, scope drift, locked-decision invariants). Use whenever the user asks for a quality pass on an epic, wants to know if an epic is buildable, asks "what's the confidence on epic X", wants to audit/score/review/check beads before dispatching a build, mentions "epic-quality-pass", or expresses doubt about whether a planned epic is ready to hand to builders. Read-only — proposes remediations as text, never modifies beads.
---

# Epic Quality Pass

## Purpose

Predict whether each bead in a given epic is buildable in one shot by a `beads-builder` sub-agent (target: ≤70k tokens of work) and whether the epic, as a whole, faithfully implements its source plan. Output is a markdown report the user reviews and applies manually.

The skill is **read-only and propose-only**. It scores, it suggests fixes, it never writes to bd.

## Inputs

A single epic ID (e.g., `PickleMatch-cpc`). Accept it whether the user passes it bare or in a sentence ("audit PickleMatch-cpc please"). If no ID is given, run `bd list --type=epic --status=open --json` and ask the user to pick one.

## Tools needed

- **Bash**: `bd show`, `bd list`, `bd dep` only — no writes
- **Read**: plan documents, CLAUDE.md, `.beads/build-history.jsonl`
- **Glob/Grep**: locate plan documents, sanity-check that file paths in beads exist

> ⛔ **Propose-only enforcement**
> Never call `bd update`, `bd create`, `bd close`, `bd dep add`, or any mutating bd command. Never use Write or Edit. The output of this skill is text in your response — the user copies suggestions into bd themselves.
> If you're tempted to "just fix" something you found, stop. The skill's value is that the user can trust its output without auditing it for unintended writes.

## Workflow

```
1. Resolve epic and child beads
2. Locate the source plan (or stop with low score if not found)
3. Read CLAUDE.md for locked decisions
4. Read .beads/build-history.jsonl if present (for empirical signal)
5. For each child bead: freshness check → rubric score (or "stale" → recommend closure)
6. Generate remediations for every bead under 95 (skip stale beads — they should be closed, not remediated)
7. Run epic-level faithfulness checks
8. Emit the report
```

### Step 1: Resolve epic and children

```bash
bd show {epicId} --json                  # epic itself
bd list --epic {epicId} --status=open --json   # children that aren't closed
bd list --epic {epicId} --status=closed --json # for context — closed beads count toward coverage
```

If the epic doesn't exist or has no children, report this and stop. An empty epic isn't auditable.

### Step 2: Locate the source plan

Try these sources in order. **Stop and report each thing you tried** — transparency matters here.

| Order | Source | How |
|-------|--------|-----|
| 1 | Epic description references | Grep the epic's description for paths like `plans/*.md`, `specs/*.md`, `../claude-code/specs/*.md`, or URLs |
| 2 | Epic design block | Check `bd show {epicId} --json` for a `design` field |
| 3 | Sibling specs | Glob `plans/**/*.md`, `specs/**/*.md`, `../claude-code/specs/**/*.md` and match against the epic title or feature name |
| 4 | Parent epic chain | If epic has a parent epic, recurse up the chain checking each parent's description and design |
| 5 | Bead-level breadcrumbs | Many bead descriptions include `**Source plan:** <path>` — collect the most-referenced plan across children |

If nothing matches: cap epic faithfulness score at **60**, emit:

```
Faithfulness: 60/100 (capped — no source plan identified)
Tried: epic.description (no path refs), epic.design (empty), specs/ (no match for "{title}"), parent chain (none), bead breadcrumbs (none).
Action: please point me at the source plan (path or URL) and I'll re-run the faithfulness check.
```

Then continue with per-bead scoring (which doesn't depend on the plan) but skip Part 3 (faithfulness analysis) of the report.

### Step 3: Read CLAUDE.md for locked decisions

Read `CLAUDE.md` once. Extract any "locked" / "do not revisit" / "Phase-0" decisions. For PickleMatch these include calm-only theme, 13+ age policy, casual-v1-only (no DUPR UI), and the SLC pilot scope. These feed into the **Phase-0 invariant check** in faithfulness analysis.

### Step 4: Read build telemetry

Read `.beads/build-history.jsonl` if it exists. Each row has `bead`, `total_tokens`, `message_count`, etc. Use it for empirical context, not as the rubric itself.

| Row count | Mode | What you do with it |
|-----------|------|---------------------|
| 0 (file missing or empty) | `static (no history)` | Pure rubric scoring |
| 1–19 | `static (N=<count>)` | Pure rubric, but cite history in the report when relevant ("a similar bead in history used 84k tokens") |
| ≥20 | `static + empirical (N=<count>)` | Same rubric, plus per-bead empirical enrichment: "your history shows beads with ≥8 ACs average X tokens" |

Note: the brief originally proposed weighting rubric categories by correlation with `total_tokens`. Real calibration requires storing the rubric score *at the time it was assigned* alongside each historical row, which we don't yet do. Until paired data exists, the empirical mode is observational only — it surfaces patterns to the user without claiming statistical rigor.

### Step 5: Per-bead scoring

For each child bead, run `bd show {beadId} --json` and inspect:

- `title`, `description`, `acceptance_criteria` (if a structured field) or the AC list parsed from the description
- `metadata.testPlanFile`, `metadata.testPlanCases`, `metadata.testPlanCoverage`
- `dependencies` (from the dep graph)
- `design` field if present
- `notes` field if present

#### Freshness check (run BEFORE applying the rubric)

Each bead description is a snapshot of the codebase at the time the bead was filed. Other work may have landed since — including work that silently satisfies the bead's acceptance criterion. If you score a stale bead with the rubric, you'll produce a confident-looking number for a bead that should just be closed.

Before scoring, verify the bead's **load-bearing claims** against current code:

- If the bead names a file that was supposed to exist or not exist, glob/Read to confirm.
- If the bead asserts a behavior is missing ("X is not wired into Y", "Z is orphan"), grep to confirm the wiring still doesn't exist.
- If the bead cites file sizes, line counts, or import counts, check the current numbers.

Then split:

- **Stale (AC already met)** — the bead's acceptance criterion is satisfied by code that landed after the bead was filed. Mark `score: N/A (stale)`, skip the rubric for this bead, and recommend closure with reason "AC already met". Cite the file path and what changed if you can pinpoint it. *This is not theoretical: in eval data, a bead claiming a hook was orphan was already wired by a sibling bead that merged 14 minutes before the epic was filed; mechanical scoring would have missed it.*
- **Drifted (claims partially out of date, AC still not met)** — the gap the bead describes is still real, but supporting details have moved (LOC changed, neighboring code refactored). Score normally, but **list the drift in a "Freshness notes" line** under the score so the builder doesn't assume the description is up to date.
- **Fresh** — claims match current code. Proceed to scoring with no special note.

Then compute the score. **Start at 100, apply penalties, floor at 0.**

#### Sizing penalties

| Signal | Detection | Penalty |
|--------|-----------|---------|
| Estimated files touched > 8 | Count file paths mentioned in description; if not enumerated, infer from AC count + scope words ("UI + API + DB" implies many) | −10 |
| Acceptance criteria count > 6 | Parse AC list; bullet points starting with "AC-N", "Given/When/Then", or numbered items | −10 |
| Cross-layer reach (n layers > 1) | Detect mentions of: UI/screen/component, API/RPC/route, DB/migration/schema, tests. Each extra layer beyond 1 | −15 each |
| Iteration headroom too thin | If estimated read-cost (existing files to read) + write-cost > ~50k tokens, no room for 2–3 retries | −10 |

#### Spec concreteness penalties

| Signal | Detection | Penalty |
|--------|-----------|---------|
| Vague AC | Phrases like "works correctly", "handles errors gracefully", "is performant" without numeric thresholds | −5 each |
| File paths missing | No file paths named for new code; not inferrable from existing project structure | −10 |
| API contract missing | New endpoint/RPC/function but no request/response shape, no Zod schema, no SQL columns | −10 |
| Test strategy missing | No mention of which tests, which file, what's asserted (and `metadata.testPlanFile` is empty) | −10 |
| Edge cases / out-of-scope unstated | No "out of scope:" or edge-case enumeration | −5 |

#### Context completeness penalties

| Signal | Detection | Penalty |
|--------|-----------|---------|
| No links to prior beads/specs/code | No bead IDs referenced, no spec paths, no existing-code pointers | −5 |
| Domain terms undefined | Project-specific jargon used without definition or reference (e.g., "the slot grid", "the report sheet") | −5 |
| Dependency graph mismatch | A bead clearly needs another's output but no `bd dep` link exists, OR a `bd dep` link is asserted but the dependency isn't real | −10 |
| Ambiguities left for builder | Open questions in description ("?", "TBD", "we should decide…") | −5 each |

#### Risk-signal penalties (additive)

| Signal | Penalty |
|--------|---------|
| New external library/API integration | −10 |
| Schema migration coupled to UI in same bead | −15 |
| Browser verification required, no test harness | −10 |
| "And/also/plus" language in title/AC suggesting two beads merged | −5 |

**For each penalty applied**, record:
- The exact phrase or absence that triggered it
- The penalty amount

The report shows this list per bead so the user can see *why* the score is what it is. Without this, the score is just a number and the user can't push back.

### Step 6: Remediations

For every bead with score < 95, propose remediations from this catalog. Each remediation must be **specific to this bead** — paste exact types, name exact files, suggest exact seams. Generic advice is worthless.

| Pattern | When | Output |
|---------|------|--------|
| **Split** | Sizing penalties dominate, especially cross-layer reach | Suggest the seam: "Split into Xa (DB+API: migration + RPC contract) and Xb (UI: screen consuming the RPC). The API contract is the stable boundary." |
| **Pin contracts** | API contract missing | Paste the proposed Zod schema / SQL columns / TypeScript types directly into the suggestion |
| **Add file map** | File paths missing | Enumerate: "Files to read: A, B, C. Files to create: X. Files to modify: Y." |
| **Add test list** | Test strategy missing | Name each test case and the file it goes in: "`__tests__/foo.test.ts`: 1) returns 401 on missing token, 2) idempotent on duplicate webhook, 3) logs failure to outbox" |
| **Add reference bead** | Bead is novel but a sibling solved the same shape | Name the sibling bead ID and what to copy from it |
| **Lift ambiguity** | Open questions left for the builder | Convert each "?" into a chosen answer with rationale, e.g., "Decision: store FK as `text` (matches profiles.id pattern), not `uuid`" |
| **Insert prerequisite bead** | Read-cost > 15k tokens just to understand existing code | Suggest a separate research/discovery bead whose deliverable is a design note the builder reads |

After listing remediations, sum the penalty amounts they would clear. If that sum brings the score to ≥95, say so. If not, surface that the bead is genuinely too big and recommend splitting even if you already suggested other remediations.

### Step 7: Faithfulness checks (epic-level)

Skip this section if no source plan was found — the report should already say so.

Otherwise:

| Check | How | Severity |
|-------|-----|----------|
| **Coverage matrix** | Read the plan; extract its requirements (sections, bullet lists, tracker items). For each requirement, check whether ≥1 child bead implements it. List unmapped requirements. | Each gap = listed in report |
| **Traceability** | Each bead description should cite the plan section it serves. Flag beads with no citation as "orphaned" — they may be in scope but the link is implicit. | Listed in report |
| **Scope drift** | Each bead should be derivable from the plan. Flag beads that introduce work the plan doesn't describe. | Listed; recommend either plan amendment or bead removal |
| **Phase-0 invariants** | For each locked decision in CLAUDE.md, scan bead descriptions and ACs for language that revisits it (e.g., re-introducing `lime` theme, adding DUPR ranking UI in v1). | Each violation = ⛔ flag |
| **Dependency sanity** | Check `bd dep` graph: are there beads serialized that could be parallel? Are there missing deps where bead B clearly needs bead A's output? | Listed with suggested fixes |
| **Non-functional coverage** | If the plan calls out a11y, perf, security, RLS, i18n, etc., is there a bead carrying each? | Each missing NFR = listed |

Compute a faithfulness score: start at 100. Each gap −5, each ⛔ Phase-0 violation −15, each scope drift bead −5, each dep mismatch −5, each missing NFR −5. Floor at 0.

### Step 8: Output format

Use this exact template. The user has tooling that may parse it.

```
# Epic Quality Pass: {epicId} — {epic title}

**Faithfulness:** {N}/100
**Source plan:** {path or "not identified"}
**Locked decisions checked:** {list from CLAUDE.md}
**Rubric mode:** {static (no history) | static (N=12) | static + empirical (N=47)}
**Generated:** {ISO timestamp}

---

## Faithfulness analysis

### Coverage
- Plan §{section} ({requirement}) → {beadId} ✓
- Plan §{section} ({requirement}) → **GAP** (no bead)
- ...

### Drift
- {beadId} adds {what} not described in plan — recommend {trim | amend plan}
- ...

### Phase-0 invariants
- ✓ Calm theme only (no `lime`/`playful` references)
- ⛔ {beadId} re-introduces DUPR ranking UI — locked to v1.1, see CLAUDE.md
- ...

### Dependency graph
- ✓ Topology reflects technical reality
- ⚠ {beadA} and {beadB} could run in parallel; remove the `bd dep` link
- ⚠ {beadC} clearly needs {beadD}'s schema but no link exists

### Non-functional coverage
- a11y: ✓ ({beadId})
- RLS: **GAP** — plan §{X} requires RLS on new tables, no bead carries it
- ...

---

## Per-bead scores

### {beadId} — {title} — {score}/100  {✓ ready | ⚠ needs work | ⛔ at risk | ◌ stale}

**Freshness:** {fresh | drifted: {one-line note} | stale: AC already met by {file/PR/commit ref}}

(For `stale`, stop here — do not run the rubric or propose remediations. Just recommend closure.)

**Penalties applied:**
- −10 (>6 acceptance criteria; counted 9)
- −15 (cross-layer: UI + API + migration in one bead)
- −10 (no API contract — `report_message` RPC mentioned but no SQL signature)
- −5 (vague AC: "handles edge cases gracefully")

**Remediations to reach ≥95:**

1. **Split into {idea-a} and {idea-b}** along the API boundary.
   - {idea-a}: migration + `report_message` RPC. Deliverable: working RPC with tests.
   - {idea-b}: ReportSheet UI consuming the RPC. Deliverable: sheet rendering, submission, success toast.
   - Estimated penalty cleared: −25 (cross-layer + AC count drops on each half).

2. **Pin the RPC contract** in the bead's design block:
   ```sql
   create function report_message(
     p_message_id uuid,
     p_reason text,
     p_details text default null
   ) returns uuid security definer ...
   ```
   Estimated penalty cleared: −10.

3. **Replace AC-7** ("handles edge cases gracefully") with two concrete ACs:
   - AC-7a: duplicate report submission within 60s returns the existing report ID (idempotent)
   - AC-7b: report on a deleted message returns 404 with no row written
   Estimated penalty cleared: −5.

**Empirical context** (if rubric mode includes empirical):
- 4 beads in your history with cross-layer reach averaged 89k tokens (range 71k–112k). This bead is wider scope than any of them.

**Projected score after remediation: 100/100.**

---

(Repeat for each bead under 95. Beads ≥95 get a one-liner: "{beadId} — {title} — {score}/100 ✓".)
```

## Critical rules

> ⛔ **Read-only**
> The skill MUST NOT call any mutating bd command, never use Write/Edit, never modify a worktree. Output is text in the response.
>
> ⛔ **Plan inference must be transparent**
> List every source you tried, even if the first one succeeded. The user needs to know whether you cheated by using a stale or wrong plan.
>
> ⛔ **Score breakdown is mandatory**
> Never report a score without showing the penalties that produced it. A bare number is unfalsifiable.
>
> ⛔ **Remediations must be specific**
> "Add an API contract" is not a remediation. "Add this exact Zod schema: `z.object({ messageId: z.string().uuid(), reason: z.enum([...]) })`" is.
>
> **Cap epic faithfulness at 60 if no source plan**
> The skill is dishonest if it confidently scores faithfulness without knowing what the epic was supposed to do. The cap forces the user to surface the plan.

## When NOT to trigger

- The user is asking for the *current state* of an epic ("how many beads are done?") — that's `bd stats` / `bd show` territory, not this skill.
- The user wants to *modify* a bead ("split this bead", "add a test list to bead X") — direct them to do it via bd or design-to-beads. This skill only proposes.
- The user wants the skill to actually fix the issues it finds. It can't. It's propose-only by design.
- The user asks for a quality pass on a single bead, not an epic. Tell them the skill operates at epic granularity; if they want one bead audited, they can wrap it in a one-bead epic or trigger the skill against the bead's parent epic.

## Tips for a useful pass

- **Sanity-check file paths before scoring.** If a bead names `src/components/ReportSheet.tsx`, glob to confirm the parent dir exists. A bead naming a path in a directory that doesn't exist gets the "file paths not inferrable" penalty.
- **Dependency mismatches are subtle.** Read the bead's description for phrases like "after X is in place" or "depends on Y" and cross-check against the actual `bd dep` graph. Implicit deps are a common source of surprise mid-build.
- **CLAUDE.md is the project's constitution.** Treat it as a hard ground-truth source for invariants. If a bead contradicts it, the bead loses, not CLAUDE.md.
- **The rubric is opinionated, not divine.** If you find a penalty rule producing nonsense for a specific bead, say so in the report ("rubric flagged X but I think the bead is actually fine because Y") and let the user decide. Mechanical scoring is a tool, not a verdict.
