# Changelog

All notable changes to git.pp-ng are recorded here, reconstructed from the
repository's own commit history. The project is pre-1.0: the fold's semantics
are still being settled (see Unreleased), and a 0.x tag asserts a working,
conformance-proven substrate — not frozen semantics.

Versioning note: this repository carried no release version line before
0.1.0. The versioned things inside it are *formats* — the record format
(FORMAT-V2), checkpoint v2, the frontier stored-file format (v: 3) — and
those keep their own numbers. The package version versions the substrate
as a whole.

## [Unreleased]

- **Retraction semantics are open.** PR #16 (a spike: dispute + settlement
  records) proposes how a record can be retracted and what a retraction
  does to derived state. It is deliberately unmerged: until it is ruled
  on, the fold has no retraction, and derived state treats every landed
  record as standing. A ruling either way is a fold-semantics change and
  will be called out here when it lands.

## [0.1.0] — 2026-11-02

First tagged release. Everything below landed on `main` between the
initial commit (2026-10-12) and this tag, and is covered by the
conformance harness at this commit: **863/863 cases identical** across
the TypeScript and Python folds, plus the worker and runner suites —
`npm test` runs all three, and CI asserts the measured counts.

### Substrate and fold

- Initial substrate (2026-10-12): task-atom records (id, parents, kind,
  effects, reason), the deterministic fold `fold(records) → state` with
  class precedence (override > task > witness), ancestry domination and
  intent-count tie-breaks; losers named and preserved, never erased.
- Depth pass (2026-10-12): fold correctness and determinism fixes,
  collection-path speedup, engineering audit.
- Memoised bitset ancestry engine (2026-10-15) replaced per-record
  ancestor sets; derived output byte-identical to the sequential fold.
- Spec-truth corrections (2026-10-14): candidates are all setters
  (DESIGN §2/§4); within-repo divergent-duplicate collection rule (§1).
- Fix #38 (2026-10-23): `__proto__` keys/ids and missing or non-string
  record ids can no longer silently drop records from the fold index;
  pinned by the PROTO-INDEX conformance family.

### Record format

- Format v2 (2026-10-15, PR #25): records are born with digests — the
  runner writes them, the Worker warns on their absence, the fold never
  reads them (digests are evidence, not input). See `docs/FORMAT-V2.md`.

### Checkpoints and delta reads

- Checkpoint v2 (2026-10-13, PR #9): closure-root pinning over the
  whole record content, so a state can be verified against exactly the
  records that produced it; a closure that changed under an unchanged
  state reports STATE-ONLY, never a silent VERIFIED.
- Checkpoint + tail derivation (2026-10-15): delta reads — derive the
  tail against a checkpoint instead of re-folding everything; identical
  tail duplicates of covered records are duplicates, not re-versions.
- Frontier checkpoints (2026-10-15, PRs #20/#21): set-form closure
  roots at the fold level, wired into the read path and the runner
  default. See `docs/FRONTIER-CHECKPOINTS.md`.

### Worker

- Cloudflare Worker coordinator on Artifacts, live at
  gitpp.superinstance.dev: derives project state from the record on
  read; no stored projection.
- Production security pass (2026-10-14): public POST /api/events
  removed, constant-time admin comparison, deep credential scrub,
  awaited dispatch with clean 500s, validated path/project parameters,
  token TTL bounds; an 18-probe security suite in `tests/`.
- Worker delta and format-v2 suites run under `npm run test:worker`.

### Runner

- `runner/pp.py`: pre-push refusals (secrets, genesis guard, kind
  discipline), refusal of empty-effects task records, and
  override-grounding — an override may only rule on keys its ancestry
  set or observed. Runner refusal/digest suites run under
  `npm run test:runner`.

### Conformance corpus

- Differential harness (TypeScript vs Python folds) grew from 839/839
  at the initial commit to 863/863, with a production-scale corpus
  (62/250/1000-record projects plus probes, three-way oracle) added
  2026-10-14.
- LEAF-BAND family (2026-10-21) pins the 1e21 leaf-number boundary,
  closing the #28 corpus hole after the boundary was ruled to stay.

### CI

- First workflow (2026-10-17): runs the suites and asserts the
  measured result counts rather than trusting exit codes.
- One-command battery (2026-10-21): `npm test` = conformance + worker
  + runner.
- 2026-11-02: the conformance job now installs bun (test:worker
  invokes it); before this fix every run passed the full 863/863
  harness and then died at `bun: not found`. CI is green at this tag.
