# git.pp — concurrent work, resolved by the record

**git.pp lets many agents (or people) work on the same project at the same time: each completion is written down as a provenance-carrying record, and the project's state is derived from that record — so competing work resolves deterministically, and nothing anyone did is ever deleted.**

For: teams running concurrent agents against shared state — config, claims, coordination data — who need one answer to "what is the state, who won, who lost, and why." Not for: resolving arbitrary code merges. git.pp resolves *labelled key conflicts* mechanically; whether two intents *mean* the same thing is a semantic question it deliberately does not settle (`RESULTS.md` §3).

## Thirty seconds: watch it decide

No install, no account — one request against the live demo project:

```sh
curl -s "https://gitpp.superinstance.dev/api/state?project=poc-quilt-1"
```

Eight records went in. Three tasks set `config.timeout` (30, 60, and 99); a reasoned override said 45. The answer (trimmed):

```json
{ "state":     { "config.timeout": 45 },
  "conflicts": [ { "key": "config.timeout", "winner": "override-1",
                   "losers": ["task-a", "task-b", "task-d"] } ],
  "orphans": [] }
```

The override's 45 stands — against a *later* push of 99 — and the losers are named, not erased: each losing version stays in its own repository, cloneable. That is the whole idea in one response: state is never stored, it is **derived**; resolution is a function of the record, and the record keeps everything.

See it live: [the project](https://gitpp.superinstance.dev/?project=poc-quilt-1) · [its records](https://gitpp.superinstance.dev/api/records?project=poc-quilt-1) · [the Desk](https://gitpp-desk.superinstance.dev), which replays the fold step by step · [the demo video (6:12)](https://github.com/patchwork-quilt/git.pp-ng/blob/main/demo/gitpp-demo.mp4), also at `demo/gitpp-demo.mp4` in this repo.

## Prove it in two commands

Everything claimed below is checkable from a cold clone — no Cloudflare account, no admin key:

```sh
(cd conformance && python3 harness.py)
python3 runner/pp.py state poc-quilt-1
```

The harness's last line is the totals line: **863/863 cases identical … — all pass**. (`npm test` runs the same harness.) The runner reads the live demo project and prints the one-line state summary: `config.timeout = 45`.

## The model (five minutes)

Five concepts, in the order they compose:

1. **Task** — the atom of work. Each task gets its own repository, forked from a shared baseline, and an agent pushes its completion there.
2. **Record** — what a completion leaves behind: one structured entry per task repo, append-only. A record has exactly five fields:
   - `id` — unique identifier;
   - `parents` — the records this one completed against (its ancestry);
   - `kind` — `task` (a completion), `witness` (an observation), or `override` (a reasoned decision);
   - `effects` — the state keys this record sets, and to what;
   - `reason` — the stated reason, present on overrides and carried in the record.
3. **Fold** — the pure function `fold(records) → state`. Project state exists nowhere else: no database, no stored projection to go stale. Any reader with the record can re-derive the state and check any other reader's answer.
4. **Conflict** — when records set the same key, the fold names a winner *and* the losers, with the ancestry that decided. Losers are preserved in their own repositories.
5. **Checkpoint** — a pin of a derived state together with the closure root of exactly the records that produced it, so a state can be verified without re-folding everything (below).

### How a conflict resolves

- **Class precedence is absolute:** `override` > `task` > `witness`.
- **Ancestry:** within a class, a candidate that is an ancestor of another is dominated and drops out; the survivors are the most-derived.
- **Ties** break toward the candidate with the most intent records (`task` + `override`) in its ancestry, then the fewest `witness` records, then the lexicographically smallest `id` — arrival order is never an input.
- **Seeing versus doing:** a `witness` carries an observation and no intent. Its uncontested observations reach state — the next agent inherits what was seen — but a witness never wins a contested key. The record distinguishes *done by us* from *seen by us*, and only one of those gets a vote.

### When the input is wrong, late, duplicated, or adversarial

- **Orphans** — records whose parents are missing, or that sit behind a cycle — are excluded from derivation and *reported*, never silently dropped.
- **Duplicated records** are deduplicated by `id`, first-seen wins. A later copy with the same `id` but different content is dropped and surfaced in `collectionWarnings` on `/api/records` — disagreement is data, not an error to hide (`DESIGN.md` §1).
- **Records may be born with a digest** (format v2, `docs/FORMAT-V2.md`): one optional `digest` field, a claim about the record's content. **The fold never reads it.** A lying digest degrades its record to v1 with a warning — never a rejection, because a rejection would hand anyone who can flip one byte a lever to make a record vanish.
- **Checkpoints verify in three verdicts** — VERIFIED, STATE-ONLY, MISMATCH — by re-deriving state and closure root from exactly the checkpoint's named closure. A tamper that touches only a record's `reason`, invisible to derived state, reads STATE-ONLY, never a silent pass (`docs/CHECKPOINT-V2.md`).

### The honest limits, early

- **Not fewer conflicts.** Conflict rates track how often the underlying work changes, not how it is resolved. The claim is resolution that is non-arbitrary, reproducible, and lossless (`RESULTS.md` §3).
- **No semantic settlement.** Labelled key conflicts resolve mechanically; meaning is out of scope.
- **Public derivation has a price, stated not solved.** One unauthenticated 62-record state read costs 126 platform operations (≈1.9¢ at documented prices); the fold is fast (below), the collection read is the cost.
- **Storage only grows — by policy.** Losing work is the audit trail; repositories in the namespace are never deleted.
- **A digest is not a signature.** Content digests authenticate *content*; they say nothing about authorship or honesty.

### Run it yourself

- `npm test` is the whole offline battery: the conformance harness, the worker suites (delta reads 50/50, format v2, security probes 24/24), and the runner suites (mechanical refusals 45/45, the digest writer) — totals in `RESULTS.md`.
- `runner/scene.sh` replays the full demo scene against a live deployment in under a minute, printing per-assertion PASS/FAIL.
- To host your own: a Cloudflare account (Artifacts requires Workers Paid), `wrangler` + Node for the Worker, and `bash`, `git`, `curl`, `python3` for the runner. `DEPLOY.md` is the runbook.

## The deep layer

Four things in this design surprised even the people building it. Each is stated plainly, with its receipt.

**1. Arrival order is never an input.** The same completions, folded in any order, produce identical state, identical conflicts, identical application order — measured, not asserted. The differential oracle compares the production engine against the legacy engine on all 863 harness cases plus 400 seeded fuzz graphs × 25 permutations: 20,000+ comparisons, zero mismatches (`RESULTS.md` §1.2). The Elixir port was re-fed its log *in reverse arrival order* and produced identical output (§1.6). Two independent judge panels still misread the prose statement of the tie-break as arrival-order-based — the mechanism was innocent, the sentence invited the error — so the seven words above now travel with the rule wherever it is stated (§2.1).

**2. Losers are never deleted — and that is load-bearing, not sentimental.** Because losing work is preserved and the state is a pure function of the record, a projection can be destroyed completely and rebuilt from the record alone. The Desk's destroy drill does exactly this: the entire object is destroyed, a fresh instance answers empty, rebuilds — and arrives at the identical state digest *and* the identical closure root (§1.5). Preservation is what makes every other claim in this README checkable by a stranger.

**3. The leaf names the stored float64.** Every record's identity in a checkpoint is the SHA-256 of a canonical form, and canonicalising a *number* is where independent implementations silently split. The rule here: an integral float64 with |v| < 1e21 is spelled as its exact integer digits — the value's own canonical name, computable identically in every language — and shortest-round-trip repr (the surface where languages disagree on padding and rounding) is confined to non-integers and the disclaimed zone. This was a considered, pinned decision that survived a real drift incident: a hub-only change to a 1e16 boundary changed the leaf of `x = 1e18` (referee `402860c3…` vs drifted hub `7d786310…`), invisibly to the then-current corpus, which carried no discriminating case. The ruling kept 1e21 (issue #28, PR #31), and a LEAF-BAND conformance family now pins the band: flipping the boundary produces 45 divergences, *all inside the family, zero elsewhere* (§1.7). The spec's own worked example was corrected in the same pass: the double nearest 7.248182946264746e20 spells `724818294626474655744`, exactly.

**4. A frontier is the union of chain-head closures.** A checkpoint over one record's ancestry barely helps a parallel project: the newest record in the 62-record scale project closes over only 21 of its 62 records, and the measured singleton-checkpoint read was ≈2.5 s — *no improvement*, reported because it happened. The frontier checkpoint — the union of all chain heads' closures, here 4 records covering 62/62 — cut the median live read from 2.42 s to ≈0.93 s (§1.4). The same section records the failure mode the design now guards: the first live frontier activation silently fell back to the full fold on every request, because a checkpoint commit's own tree served its already-covered record as "tail" and tripped the disjointness tripwire. The fix — a tail record deep-equal to the closure's copy is a duplicate serving and is dropped; a *divergent* copy still forces the full fold — is pinned by tests that fail without it.

### Measured numbers

| What | Number | Receipt |
|---|---|---|
| Conformance (TS vs Python referee) | 863/863 cases; 1,029 checkpoint probes; 3,231 witness-freshness checks; 16,290 closure-root digests; 16,290 delta-vs-full derivations; 2,812 oracle checks | `conformance/harness.py` |
| Frontier family | 16/16 cases | `RESULTS.md` §1.4 |
| Format v2 family | 19/19; 1,758 born-ignored derivations byte-identical; root assembly 16,436 == recomputed | `RESULTS.md` §4 |
| LEAF-BAND family | 12/12 cases; 29 leaves + 29 roots pinned as literals | `RESULTS.md` §1.7 |
| Fold speed, 62 records | 7.3 ms legacy → 1.8 ms bitset engine (Node, median) | `RESULTS.md` §1.2 |
| Fold speed, 5,000 records | 2,712 ms legacy → 110 ms | `RESULTS.md` §1.2 |
| Three-way agreement (TS == Python == brute-force oracle) | identical at 62 / 250 / 1,000 / 2,500 / 5,000 records | `RESULTS.md` §1.3 |
| Live read, 62-record project | 2.42 s full fold → ≈0.93 s with frontier checkpoint (medians) | `RESULTS.md` §1.4 |
| Third port (Elixir/BEAM, written from the docs alone) | 898/898 derive · 1,056/1,056 verify · 3,259/3,259 freshness · 16,574/16,574 roots on the current corpus | `RESULTS.md` §1.6 |

### The verification contract

You do not have to trust any of this. The conformance corpus in `conformance/` is the contract: two implementations in this repo (TypeScript `src/fold.ts`, Python `referee/fold.py`) pass it byte-identically, a brute-force oracle checks maximality independently, and a third implementation in another language (below) was written from `DESIGN.md` and the docs alone and passes the same corpus. `RESULTS.md` is the ledger — what was measured, what independent readers said, what is **not** claimed — and it corrects this README's numbers if they ever diverge. Design depth: `DESIGN.md`; `docs/CHECKPOINT-V2.md`; `docs/FRONTIER-CHECKPOINTS.md`; `docs/DELTA-READS.md`; `docs/FORMAT-V2.md`; `docs/ENGINEERING.md`; `docs/CUTTING-EDGE.md`; open doctrine questions are kept in `docs/OPEN-QUESTIONS.md`, labelled as open.

## The shelf — sibling tools, one repo each

- **[gitpp-action](https://github.com/patchwork-quilt/gitpp-action)** — this fold as a GitHub Action: derived state as an advisory job summary inside your own CI. Start here if you want the verdict without installing anything.
- **[gitpp-app](https://github.com/patchwork-quilt/gitpp-app)** — the GitHub App: webhooks in, the derived state back as advisory Checks on pull requests.
- **[outbound-scanner](https://github.com/patchwork-quilt/outbound-scanner)** — a pre-send fence for agent harnesses: catches session secrets in outbound replies before they ship. Unrelated to the fold; same programme, same honesty rules.
- **[git.pp-desk](https://github.com/patchwork-quilt/git.pp-desk)** — the fold replayed step by step in the browser: step the record, watch state derive, test arrival orders.
- **[git.pp-grid](https://github.com/patchwork-quilt/git.pp-grid)** — coordination surfaces (custody, on-call) built on the same record idea, for humans.
- **[gitpp-beam](https://github.com/patchwork-quilt/gitpp-beam)** — the fold re-implemented in Elixir/BEAM from the documents alone; the portability proof and the corpus's third port.

## Architecture, in one breath

Portable git carries the record. A Cloudflare-shaped Worker forks task repos, coordinates tokens and events, gathers the record set, and serves derivations — it never decides the state. The fold decides the state. Push events feed a Queue for observability; derivation is per-request, so a served state is never a stale projection.

## License

MIT.
