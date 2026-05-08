# Changelog

All notable changes to this plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] - 2026-05-08

### Added
- Initial public release as a unified toolkit.
- **`design-to-beads` skill** — lossless conversion of design docs / PRDs / task lists into sized beads epics with validated dependencies and triple subagent review.
- **`epic-quality-pass` skill** — read-only audit that scores each child bead on a 0-100 buildability rubric and proposes remediations. Verifies the epic collectively implements the source plan.
- **`build-dispatch` skill** — pure-dispatcher orchestrator that picks up the next ready epic, runs a parallel TDD pipeline per bead, and ships one PR per epic.
- **Four sub-agents** powering build-dispatch: `beads-tester`, `beads-builder`, `beads-validator`, `beads-merger`.

### Notes
- This release supersedes the standalone `cstaulbee/build-dispatch` and `cstaulbee/design-to-beads` repos. Those repos are deprecated; new installs should use this toolkit.
