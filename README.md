# beads-toolkit

**A three-stage autonomous build pipeline for [Claude Code](https://claude.com/claude-code), built on [beads](https://github.com/steveyegge/beads).**

You hand it a design doc. It turns the doc into a sized epic, audits the epic for buildability, dispatches parallel TDD pipelines per bead, and ships one PR per epic — without supervision.

```
   design doc / PRD                 ship-ready PR
        │                                  ▲
        ▼                                  │
 ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
 │ design-to-beads│─▶│epic-quality-   │─▶│ build-dispatch │
 │  (decompose)   │  │  pass (score)  │  │  (parallel TDD)│
 └────────────────┘  └────────────────┘  └────────────────┘
                              │
                       remediations
                       (text only)
```

This plugin bundles all three skills plus the four sub-agents that power the build pipeline.

---

## What's in the box

### `design-to-beads` skill
Convert a finalized design doc, PRD, or markdown task list into a fully-structured beads epic. Enforces hard sizing limits per bead (≤5 files, ≤6 acceptance criteria, ≤300 words, single domain), validates the dependency graph for maximum parallelization, and runs three independent subagent review passes before any `bd create` runs.

**Trigger phrases:** *"convert this design doc to beads"*, *"break this PRD into beads"*, *"plan this feature in beads"*.

### `epic-quality-pass` skill
**Read-only.** Scores each child bead in an epic on a 0-100 buildability rubric (sizing, spec concreteness, context completeness, risk) and proposes specific remediations to elevate every bead to ≥95. Also verifies the epic collectively implements its source plan (coverage, traceability, scope drift, locked-decision invariants). Outputs a markdown report you apply manually — never modifies beads.

**Trigger phrases:** *"quality pass on epic X"*, *"is this epic ready to build?"*, *"score the beads in epic X"*.

### `build-dispatch` skill + 4 agents
Pure-dispatcher orchestrator. Picks the next ready epic, dispatches a per-bead **Tester → Builder → Validator → Merger** pipeline in parallel, gates each merge with a typecheck, runs full quality gates on the assembled epic, and opens one PR. The orchestrator itself never touches `git` or `npm` — every state-changing action goes through one of the four sub-agents (`beads-tester`, `beads-builder`, `beads-validator`, `beads-merger`).

**Trigger phrases:** *"pick up the next epic"*, *"start the build"*, *"run the build pipeline"*.

## Prerequisites

1. **Claude Code** — [install it](https://docs.claude.com/en/docs/claude-code/overview) if you haven't.
2. **`bd` (beads) CLI** — system-wide:
   ```bash
   curl -fsSL https://raw.githubusercontent.com/steveyegge/beads/main/scripts/install.sh | bash
   ```
   Then `bd init` in your project. See the [beads docs](https://github.com/steveyegge/beads).
3. **A project that supports `npm run typecheck`, `npm run lint`, and `npm run test`** — required for `build-dispatch` only. The other two skills work independently. If your project uses different scripts, alias them in `package.json` or fork the merger agent.

## Install

```
/plugin marketplace add cstaulbee/beads-toolkit
/plugin install beads-toolkit@beads-toolkit
```

Updates: `/plugin update beads-toolkit@beads-toolkit`.
Uninstall: `/plugin uninstall beads-toolkit@beads-toolkit`.

## Quickstart — end-to-end

The full pipeline, from design doc to PR:

1. **Drop your design doc into the project** (e.g. `docs/feature-x.md`).
2. **Convert it to an epic.** Tell Claude: *"Convert `docs/feature-x.md` to beads."* The `design-to-beads` skill triggers, drafts the epic, runs three subagent reviews, and creates the beads when reviews pass.
3. **Audit the epic.** *"Quality pass on epic X."* The `epic-quality-pass` skill produces a buildability report. Apply any remediations it suggests.
4. **Ship it.** *"Pick up the next epic."* The `build-dispatch` skill takes over, dispatches parallel TDD pipelines, and opens a PR when the epic is fully merged and gates pass.
5. **Review the PR** on GitHub.

You can also use any skill standalone. Convert without dispatching. Audit without converting. Build without auditing. The pipeline is recommended but not required.

## CLI flags (build-dispatch)

Pass via the skill prompt — e.g. *"start the build with --builders 3"*.

| Flag | Default | Description |
|------|---------|-------------|
| `--tdd` | `true` | TDD-first pipeline (Tester → Builder → Validator). Disable to skip Tester. |
| `--abort-threshold` | `50` | Abort epic if more than N% of resolved beads fail (0-100). |
| `--builders` | `5` | Max concurrent bead pipelines. |
| `--timeout` | `600000` | Per-stage timeout in milliseconds. |

## Sizing rules (design-to-beads)

Hard caps per bead. Anything bigger gets split:

| Limit | Cap |
|-------|-----|
| Files to create/modify | ≤5 |
| Acceptance criteria | ≤6 |
| Files needing exploration | ≤10 |
| Description length | ≤300 words |
| Module/domain scope | 1 |

Common automatic splits: full-stack slices → backend bead + frontend bead; CRUD bundles → per-operation; cross-module work → per-module-boundary.

## Buildability rubric (epic-quality-pass)

Each bead scored 0-100 across:

- **Sizing** — does it fit a Sonnet builder's ~120K usable context?
- **Spec concreteness** — are acceptance criteria testable, unambiguous, locally checkable?
- **Context completeness** — does the bead include or point at everything a fresh agent needs?
- **Risk** — locked decisions respected? Hidden dependencies? Cross-bead coupling?

Target: every bead ≥95 before dispatch. Below that, the report names what to fix and how.

## Known limitations

- **`npm`-only builders.** The `beads-merger` agent calls `npm run typecheck|lint|test`. Yarn/pnpm/bun/non-JS users need to alias scripts in `package.json` or fork the merger.
- **Beads-only.** No GitHub Issues / Linear / Jira adapter.
- **Worktree-heavy.** `build-dispatch` disk usage scales with `--builders`; each bead gets its own worktree.
- **Opus-default agents.** Each bundled agent specifies `model: opus`. Edit the agent frontmatter to switch to Sonnet/Haiku per stage if cost matters more than throughput.
- **English-language docs assumed** for `design-to-beads` heuristics.

## Contributing

Issues and PRs welcome. Please open an issue before sending a large PR. Sizing-rule changes in `design-to-beads` should include a worked example. Agent-contract changes in `build-dispatch` need a real-epic smoke test in the PR description.

## Migrating from older standalone repos

The previous `cstaulbee/build-dispatch` and `cstaulbee/design-to-beads` plugins are deprecated. Migrate with:

```
/plugin uninstall build-dispatch@cstaulbee-build-dispatch
/plugin uninstall design-to-beads@cstaulbee-design-to-beads
/plugin marketplace remove cstaulbee/build-dispatch
/plugin marketplace remove cstaulbee/design-to-beads

/plugin marketplace add cstaulbee/beads-toolkit
/plugin install beads-toolkit@beads-toolkit
```

## License

MIT — see [LICENSE](LICENSE).
