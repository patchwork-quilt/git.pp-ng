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

## [0.2.0] — 2026-11-02

Retraction is fold law. Casey's ruling of 2026-11-02 adopted PR #16
(dispute + settlement) **with the contested amendment**, under the
governing constraint that the fold says only what the records can back
up. Two further rulings from the same packet are recorded in this
release; neither changes fold behaviour.

- **Dispute + settlement records.** The fold is now two passes: a
  settlement pass computes the retracted set from the record set alone
  (a settlement bites only if its author is the target's author or
  captain-class — the author of an applied override; anonymous
  settlements never bite; open/rejected/unauthoritative disputes are
  exactly inert), then the classic derivation runs over what stands. An
  upheld dispute voids the target's effects; the target keeps its graph
  position and order slot, lends no cleanliness, and its descendants are
  unaffected except through derivation itself. Full semantics:
  `docs/RETRACTION-SPIKE.md`; DESIGN.md §2 states the law in brief.
- **The contested amendment.** A retraction the target's author neither
  filed nor settled — a ratified third-party accusation — is marked
  **contested** on the retraction surface
  (`retraction_status` / `retractionStatus` gains `contested` entries:
  `{record, dispute, settlement, disputed_by, settled_by}`), never a
  silent void. Self-filed or self-settled retractions are never marked.
- **Frontier checkpoints range over maximal landed records** (was:
  maximal applied). A settlement is typically a leaf meta record; the
  applied-only frontier left landed rulings outside every closure and
  pinned the unretracted state. The theorem is restated as
  C == landed set (`docs/FRONTIER-CHECKPOINTS.md`); sets without meta
  records — including the frozen poc-scale-1 frontier — are unchanged.
- **STATE-ONLY stands** (checkpoint closure differences stay
  informational, never accused as MISMATCH). The ruling, its date, and
  tailor-fit guidance are recorded in code comments at the verification
  decision point in both ports.
- **Lock sovereignty DECIDED (A)** (`docs/OPEN-QUESTIONS.md` §1d):
  override always wins; locks constrain defaults, not deeds; a
  must-bind policy is modelled as an override. Design-only; no code
  models scoped defaults yet.
- Conformance at this tag: **1155/1155 cases identical** across the two
  folds (863 pre-law cases + 34 retraction cases + 258 sprinkled
  inertness variants), 1161 checkpoint probes, 302 retraction statuses
  and 8 contested markings pinned, plus the worker and runner suites —
  `npm test` runs all three, and CI asserts the measured counts.

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
