# Blog review — 2026-07-19

> **Status (later on 2026-07-19): all fixes from this review have been
> applied to the posts.** This file is kept as the pre-fix record; line
> numbers refer to the versions reviewed, not the current ones. Two
> non-post items were left alone on purpose: committing/pushing is the
> author's call (including the untracked `shy-heap.md`), and the
> Görgens/Schorer sign-off question is a personal decision, not an edit.

**This file lives at the repo root, outside `content/` and `static/`, so Zola
never sees it and it cannot be published.** Delete or `.gitignore` it if you
don't want it committed either.

Scope: all 8 posts in `content/posts/`, plus site mechanics. Every concrete
claim in each post was cross-checked against the relevant code in `~/prog/`
(tapecheck, proptest, splittable_random-upstream, bcachefs\*, ktest,
TwoTimePad\*, miden-collision, mozak, shy-heap, shy-heap-lean, offline-heap)
and against the online sources cited (GitHub issues/PRs, arXiv, Wikipedia,
the Wayback Machine, the Equilibrium Aleo article). `zola build` passes clean
(8 pages, internal links OK).

---

## Overall verdict

This is a strong blog. The dominant quality is *evidence discipline*: nearly
every number, quote, and error message in the posts traces to a real artefact
— saved benchmark outputs, commit messages, upstream issues — and the direct
quotes checked (Bobbin Threadbare, Edward Kmett, the OxCaml mode error, the
Equilibrium Aleo quote, the Miden TODO comment) are verbatim or honestly
ellipsized. The self-critical framing (the Miden scope-up-front, the
two-time-pad autopsy, the Mozak apology) is genuine and earned.

The errors that exist cluster in three predictable places:

1. **Numbers that went stale as the code kept moving** after the post text
   was written (89→113 choices, "byte-for-byte", the Docker script).
2. **Compressed technical summaries that blur which change contained what**
   (GFP_NOFS in the SRCU fix, `mapping_set_gfp_mask`, dm-delay vs throttling).
3. **Scholarly citations written from memory** (Erickson, Fife–Oxley) — the
   two worst errors in the whole blog are here.

One systemic gap: `frozen-ls.md` and `borrow-checker-review.md` contain
**zero URLs** despite describing Reddit threads, GitHub issues, PRs, and ktest
tests that all exist. The other six posts link well. Links are what let a
reader distinguish your claims from your memory; add them.

Also be aware: commit `d940b28` ("Fix errors found in a full review of all
five posts") shows a previous review pass happened. This pass found more,
mostly in material added or changed since.

---

## Per-post reviews

Posts are ordered by severity of findings, worst first.

### 1. `borrow-checker-review.md` — What the borrow checker won't review

**Verified:** the deadlock trace matches commit `c1fa393fc5d2`'s message
almost verbatim; the dedup pending-flag livelock mechanism is real and still
present in `fs/bcachefs/data/dedup.c` (mismatch path returns without clearing
`dedup_pending`); the Verus proofs (`verus-proofs/eytzinger_verify.rs`,
`bpos_verify.rs`) exist upstream, authored by Kent/ProofOfConcept — so
"bcachefs already carries" is accurate and doesn't overclaim ownership;
`decreases` clauses match Verus docs; the effects/ownership taxonomy is
sound.

**Wrong / overstated:**

- **L35–40, the central mm claim — the paragraph doing the most technical
  work in the post is wrong for the exact case the post is about.**
  "Swap writeout from reclaim is gated by `__GFP_IO`, not `__GFP_FS`, so
  `PF_MEMALLOC_NOFS` … does nothing to stop reclaim issuing a swap write" —
  this ignores upstream commit `d791ea676b66` (v5.19, NeilBrown, "mm: reclaim
  mustn't enter FS for SWP_FS_OPS swap-space"): `may_enter_fs()`
  (`mm/vmscan.c:1079–1093`) returns false for a swapcache folio with only
  `__GFP_IO` **when `SWP_FS_OPS` is set** — and bcachefs's swap sets
  `SWP_FS_OPS` at swapon. So for swap-on-bcachefs, a NOFS allocation *does*
  stop reclaim from re-entering via swap writeout. Your observed deadlock ran
  through `GFP_KERNEL` (both flags set), where the distinction is moot, and
  NOIO remains the right conservative fix (it also covers swap-slot
  allocation and non-swapcache paths) — but the stated *reason* will not
  survive review by an mm person. Fix: acknowledge the `SWP_FS_OPS`
  carve-out and frame NOIO as belt-and-braces, or demonstrate a
  NOFS-permitted recursion path empirically (your ktest harness can do this).
- **L81 "the filesystem's journal simply freezes, live-locked"** — the
  reprocessing loop is verified, but your own investigation docs record a
  kernel *panic* (the relock bug, fixed in `e10edaec8293`) and dmesg spam as
  the observed failure modes; no documented journal freeze. Safer: "the
  reconcile queue never drains".
- **Description vs body:** "Two bcachefs bugs that cost me a week each" vs
  "I spent last week" / "ate my week" (singular). Pick one.

**Nits:** L113 "a shrink engine around a mode checker" is a cryptic
self-reference that will lose most readers; L81 "under the right input" could
name the input (same-size extents with colliding checksums on a dedup-enabled
fs).

### 2. `frozen-ls.md` — How a frozen `ls` turned into swap on bcachefs

**Verified — and remarkably so:** the crash setup, device counts, memory
figures, the SRCU fix (`af6e9a90935c`, 25 sites `bch2_trans_unlock()` →
`bch2_trans_unlock_long()`, the `drop_locks_long_do()` macro), the
`time ls` result, Kent's "unlock, block, then unlock long" shape (his own
removed `bch2_fsck_ask_yn()` workaround did exactly that), the POC
AI-assistant detail (corroborated independently: upstream commits by
"ProofOfConcept", poc.bcachefs.org, The Register 2026-02-25), the swapfile
rejection story (`generic_swapfile_activate` "holes" error; `SWP_FS_OPS`
wired in `vfs/swap.c`/`vfs/fs.c`), the initramfs shadowing bug, the GFP
audit commits, the ktest `swap-stress` branch (`swapfile_deadlock.ktest`
etc.), and the version-1.4 anecdote (`member_seq` first shipped in v6.8).

**Wrong / overstated:**

- **L116 "`mapping_set_gfp_mask` on the swap file's address space"** — does
  not exist in `fs/bcachefs` on any branch, nor in its history
  (`git log --all -S` empty). The actual reclaim armor is
  `memalloc_noreclaim_save()` in `bch2_swap_rw`, btree-node pinning, the
  16 MB btree-cache pre-reserve, disk reservation, and the NOIO audit. Delete
  the clause or name the real mechanisms.
- **L88 "QEMU-plus-`dm-delay` contraption … fake slow 50 ms disks"** — the
  QEMU harness (`run-vm-test.sh`) used blkio write throttling
  (512 KB/s), never dm-delay; "50 ms" appears in no local artefact. dm-delay
  exists only as an optional knob in the later ktest test. The detail belongs
  to the wrong phase of the story.
- **L63–65: the `ls` fix "…using `GFP_NOFS` in the right spots"** — the SRCU
  series contains zero GFP-flag changes; it's purely unlock → unlock_long.
  This conflates the later swap work.
- **L54 "bcachefs takes one in the btree commit path"** — the SRCU read lock
  is taken per btree *transaction* (`bch2_trans_begin`), reads included.
- **L122 shrink chronology** — jullanggit's series starts 2026-02-16, six
  days *before* your single shrink commit (2026-02-22). "I'd taken a first
  swing at [it] before" isn't supported by visible git dates; soften to
  "around the same time" unless you have off-git evidence.
- **Attribution gap:** the upstream artefacts (bcachefs-tools PR #646, ktest
  PR #63) are posted under Komzpa's account with dual sign-off. The solo
  framing ("I wrote the patches") omits the co-signer — a reader who clicks
  the PRs will trip over this. Related, and worth a conscious decision: the
  sign-off inside commits authored "Matthias Goergens" reads "Matthias
  Schorer <matthias@schorer.dev>".
- **No links.** This post references Reddit threads, GitHub issues (#934,
  #636), PRs, and ktest tests — link them.

**Unverifiable:** Reddit 403s from this environment, so Kent's exact words,
the POC "first run" remark, and the quoted "stress test setup was the only
reason" line couldn't be checked (circumstantial corroboration is strong).

**Nits:** L57–58 antecedent muddle — per your own INVESTIGATION.md it's the
*background* threads that hold SRCU while blocked on HDD I/O, and the
foreground blocks on the btree locks they monopolise; "thirty, sixty seconds"
(L38) vs "multi-minute freeze" (L59) in adjacent sections.

### 3. `shy-heap.md` — The shy heap

**Verified:** the algorithm's arithmetic all checks out (primal certifies
≥ r−εm, dual ≥ k−εm, ≤ 2εm unresolved per round; ε=1/3 → settle ≥ 1/3;
geometric series sums to 3×). The Rust repo is public as `shy-heap-rs` and
contains the actual two-pass algorithm (`src/schubert.rs`: `linear_loop`,
`dualise_ops` with no key comparisons) and `src/pairing.rs` ("Soft heaps
based on pairing heaps"). The paper draft (`paper/paper.tex`) ends
mid-argument exactly where the post admits. The Lean claims are real:
`offlineSurvivors_eq_exact` (`ExactReduction.lean:190`),
`ObservableOneThirdSoftHeap` as the black box, the two-thirds recurrence in
`Complexity.lean`, `canonicalMatroid` — and **zero `sorry`/`axiom`/`admit`**
in the project (I did not run a build, so "compiles green" is unverified).
arXiv 1802.07041 is indeed Kaplan–Kozma–Zamir–Zwick, and §2 is the fair seed
of the set-valued API. All eight external links return 200. The "carried for
years" claim holds (C++ soft-heap repo from 2016, offline-heap notes 2023).

**Wrong / overstated:**

- **L42–44: the Erickson citation is wrong.** In the current edition of
  Erickson's *Algorithms* the word "matroid" does not occur, and the
  unit-time/deadline/penalty scheduling exercise is not there. The canonical
  source is **CLRS §16.5, "A task-scheduling problem as a matroid"** — same
  costume, same greedy. Cite CLRS or drop the attribution.
- **L115–117: "near-synonyms … carry conflicting definitions" misdescribes
  the linked paper.** Fife–Oxley state that nested = freedom = generalized
  Catalan = shifted = Schubert (all *coincide*, due to Crapo), with
  nested ⊊ laminar. The names are redundant, not conflicting — and Theorem
  1.4 in that very PDF characterizes exactly your class. Coining "heap
  matroids" is fine; the stated justification contradicts its own link.
- **L110–112: the matroid sentence needs the fixed-trace quantifier.** With
  the trace free, "insert x / delete anything" reaches every subset (the free
  matroid); the interesting statement fixes the trace (deletes fixed in
  position, free in target), which is what the Lean proof does
  (`canonicalMatroid ops`). One-clause fix: "Fix a trace — inserts and
  deletes in a fixed order — and let each delete remove anything."
- **L137 "bridged to Mathlib's maximum-weight-basis machinery"** — the
  max-weight-basis predicate is a *local* abbrev
  (`MatroidMinorObjective.lean:166`); Mathlib contributes the Matroid/IsBase
  API. Say "bridged to a maximum-weight-basis predicate over Mathlib's
  Matroid library".
- **L132 "a family of new soft-heap implementations based on pairing
  heaps"** — publicly thin: only `pairing.rs` is pushed; `lazy_pairing.rs`
  and `catenable.rs` sit in 8 unpushed local commits. One implementation is
  public, not a family.

**Nits:** add "comparison-based" to the opening theorem claim (integer
priority queues — vEB, Thorup — beat log n); the "for fixed ε" qualifier
belongs at the first "amortised constant time" (L13), not L77; "total work
three times one linear pass" hides that each round is *two* soft-heap passes;
the heaviest-balanced-parentheses aside (L125) is a private reference without
a citation; "reviewer-ladder series" (L141) retroactively coins a series name
no prior post uses.

### 4. `choice-tapes.md` — Your generators already know how to shrink

**Verified:** `let int = atomic` quote and the int32/int64/float/char/bool
claim; the `int_inclusive` 5/5/90 weighted union; the `list_generic`
draw-order finding; all six table rows including the worst-case values and
avg call counts (they match `design/shrink-table-results.txt` verbatim);
`[100]` in 49 calls; the replay-under-different-seed test (ran the prebuilt
binary: passes); `Tape_test` drop-in API; the OxCaml 10129-attempt
determinism claim; proptest PR #658 and splittable_random PR #2 exist and
match their descriptions; the Conjecture characterization matches the
Hypothesis article; QCheck2/Bam integrated-shrinking claims are fair (Bam's
own docs concede the bind problem); the `fn` aside matches
`design/stream-keyed-tapes.md`.

**Wrong / stale:**

- **L75–76 "byte-for-byte unmodified"** — false, and your own README says so:
  "unmodified except the dune file … and one portability fix in
  generator.ml". Adopt the README's phrasing. Ironic in a post whose whole
  point is evidence.
- **L73 "89 typed choices"** — stale. Commit `39da33e` says verbatim the tape
  "grows from 89 to 113 choices because Log_uniform's inner draws are now
  recorded individually"; the current test prints 113.
- **L126 "one failing case in ten starts inside a trap"** — 10% of *draws*
  hit constant branches, but `lo`-branch cases are already minimal for a
  threshold property; only the `hi` branch (~5%) actually traps.

**Nits:** the table omits that stock's call count is ~0, hiding the cost side
of the trade-off; "in one attempt" (L115) → "on its first proposal"; "these
dozen functions" is loose (≈10 pre-seam, 6 hooks now); QCheck2/Bam are
discussed qualitatively but never measured — the 0/100-vs-100/100 comparison
against stock, which punts by design, reads bigger than it is.

### 5. `mode-checker-review.md` — The mode checker reviewed my code

**Verified:** the global `Tape.t option ref` exists at parent commit
`e38012f`; the probe and the mode error match `blog/materials/` character for
character; the refactor is verbatim commit `8773745`, literally one commit
after the global, same afternoon; `Modes.Portable.t`, both `Domain.spawn`
alerts, and `Domain.Safe.spawn : (unit -> 'a) @ portable once -> 'a t` all
match OxCaml stdlib; all eight benchmark numbers match the saved materials;
the 4.6x/4.8x/12% arithmetic is right; the postscript's design question is
really in PR #2 ("raised … before any human saw the patch" — the PR body
says it).

**Overstated:**

- **L107 "Attempt counts are identical at every domain count"** — true for
  this benchmark, stated as a design property; your README contradicts it
  generally ("with a pool the engine evaluates batches speculatively, so
  attempt counts can exceed the sequential run's"). Scope it to the
  benchmark.
- **L130 "Flambda2 runs the sequential engine 12 percent faster"** — the
  comparison confounds Flambda2 with compiler version (stock 5.3.0 vs OxCaml
  5.2.0+ox) and rests on single-run wall times. The number is real; the
  attribution isn't isolated.

**Nits:** the error and alert quotes silently abridge (file/line locations
dropped, "(the CPU core count)" removed) without ellipses — the materials
files have the verbatim originals, so link or footnote them; "the diff is the
honest one" — the after-commit also contains the pool and benchmarks; "I had
already written the comment apologising for it" — no apologetic comment
exists at `e38012f` (rhetorical flourish, your call).

### 6. `proof-of-0-equals-1.md` — A proof that 0 = 1, in a real zk-VM

**Verified:** everything quotable. December 2022 timing (commits and issue
#605 both 2022-12-21); the MASM program; the teaser-style report; Bobbin's
reply ~15h later, quote verbatim, linking exactly the TODO line at
`join_block.rs`; the TODO comment byte-for-byte; Kmett's `Hash(3a+1, b)` /
`Hash(5a+2, b)` and the affine-over-multiply rationale, verbatim; PR #682's
domain-in-second-capacity-element mechanism; the RPO migration already in
flight. The honest scoping up front ("the Miden team already knew") is
exemplary.

**Wrong / stale:**

- **L144–146 "RPO's wider state left a capacity register genuinely free"** —
  wrong mechanism. RP and RPO as used by Miden have identical state geometry
  (12×64-bit, rate 8, capacity 4). What changed: RP's `merge` consumed a
  capacity element for input length; RPO zeroes the capacity and reserves
  element 1 for a domain, backed by a security argument (RPX spec,
  ePrint 2023/1045) that spending it costs nothing below the 128-bit target.
  Fix: "RPO's redesigned padding scheme freed the second capacity element,
  and its security analysis showed spending it costs nothing below the
  128-bit target."
- **L137–139 "the hash function then in use had none to spare"** —
  overstated: in the thread, domain-in-capacity under RP was Bobbin's
  *preferred* option, priced at "reduce security by 1 bit" — not impossible.
  The "no free slots" objection was about rate/input slots, against widening
  the hash input.
- **L203 "the Dockerised proof you can run yourself"** — stale as of the
  post's own date: the script clones `miden-vm --branch next` unpinned, and
  `next` is now a Poseidon2-era workspace with neither the `--features=executable`
  CLI nor the `compile/run/verify` subcommands. The script fails at the first
  cargo invocation, and an RP-era proof could never verify against a post-fix
  verifier anyway. Pin the commit or date-stamp the claim.
- **L157 "a fresh prime for every new node type"** — primality is irrelevant;
  distinct nonzero multipliers suffice.

**Nits:** ekmett's cost figure was "per instruction"; the post's "per node"
silently re-bases it (defensible, be aware); the ellipsis in the Bobbin quote
drops "not too difficult to do right now" — a clause that *strengthens* your
thesis, free to restore; noop/no-op inconsistency.

### 7. `two-time-pad.md` — The two-time pad wanted a 5-gram, not a neural net

**Verified:** the archived puzzle page resolves and matches (46-char alphabet
with exactly 9 punctuation marks, mod-46, "created September '04, retired
December '06"); the OTP math is stated correctly; all four branch names
(`fractal`, `frac-again`, `truncated`, `more-loss`) exist; the entire
architecture-search litany checks out in code (PReLU/dense residual, GRU,
SimpleRNN, DenseNet skips, 2-layer LSTM with BatchNorm, mixed precision,
150-char window, clipnorm 0.5, LR schedules); the mmap/pipe Rust
preprocessor; the "77.6% improvement" commit is verbatim; the 5-gram model
(order-5 table, 24,452,581 corpus chars, Witten-Bell), the beam mechanics,
the 97%/beam-4000 claims, and both recovered plaintexts (Mayor of
Casterbridge ch. 1; Voyage of the Beagle Galápagos chapter) all match the
repo — and neither Hardy nor Darwin is in the corpus list.

**Wrong / imprecise:**

- **L53 "17 GB of Project Gutenberg"** — the actual corpus is
  14,938,077,525 bytes ≈ 13.9 GiB / 14.9 GB; the repo's own docs say
  "16 GiB". Say "~15 GB" or match the docs.
- **L82 "a couple of dozen public-domain books"** — `fetch.sh` lists 35.
- **L13 "every plaintext of the right length is exactly as likely as any
  other"** — only under a uniform plaintext prior; the correct statement is
  that the ciphertext is independent of the plaintext. Given the very next
  paragraphs hinge on English being non-uniform, a pedant will notice.
  Suggest: "the ciphertext alone tells you nothing — any plaintext of the
  right length could have produced it."
- **L88 "about 97% of the characters of ordinary prose"** — mild
  cherry-pick: 97% is 19th-century prose; the README's full-suite mean is
  96.8% with a 93.2% minimum on modern web fiction. And "errors are almost
  all proper nouns and digit runs" is slightly overstated — the shipped
  outputs also garble ordinary words.

**Nits:** Witten-Bell is interpolation, not "back off" (one word: "blend");
the archive link lands on the top of a multi-puzzle page — tell readers to
scroll to "Decrypting the Two-Time Pad"; "seven seconds" was ~9s on this
machine ("about" covers it); "two years" is narrative license — the repo
spans 2019→2026 in bursts.

### 8. `solution-in-search-of-a-problem.md` — A solution in search of a problem

**Verified:** `cast_list` is literally in the code (336 occurrences, 75
files); the script contents ("who called whom, with what arguments, what came
back") match `CrossProgramCall { caller, callee, argument, return_ }`; cast
members named by code hash (`ProgramIdentifier(Poseidon2Hash)`); the
verifier/cast-tree binding; the token/wallet example; recursion used for
compression (batch STARK wrapped in one plonky2 proof); RV32IM Rust guests;
the repo is archived read-only (Oct 2025); the Equilibrium quote is exact
and the Aleo characterization is fair. The one-program-world diagnosis is the
strongest section and is fairly argued — it concedes the interpreter model's
virtues instead of strawmanning them.

**Wrong / overstated:**

- **L56 "a token program insists transfers conserve tokens"** — not in the
  code: the example `transfer` only reassigns ownership of a whole
  `TokenObject`; `Mint`/`Burn`/`Split` are commented out. Drop conservation
  or say "insists the sender's wallet approves" (which the code does
  enforce).
- **L57 "the relevant wallets agree"** — only the *remitter's* wallet is
  consulted; the remittee's pubkey is just written into the object.
- **L72/L152 "must name the wallet's verification key at authoring time"** —
  slight simplification: the hard constraint is the inner circuit's shape
  (`CommonCircuitData`); the VK itself can be a proving-time witness in some
  schemes. The closed-world conclusion survives; "must bake in the wallet's
  circuit" is the nitpick-proof phrasing.
- **L153 "one trusted executor"** — wrong word: zkRollup executors are
  emphatically *not* trusted for validity (that's the point of the proof).
  You mean "single, all-seeing".
- **L144 "a program reveals neither its code nor its private inputs"** —
  stated in present tense as a design property, but no ZK blinding machinery
  exists in `circuits/src`, and the repo's README disclaims security. "The
  prototype was never finished" partially covers this; make the hedge
  explicit where the claim is made.

**Nits:** L85 "with separate scripts" smuggles in an unintroduced composition
mechanism — the post established one shared script per joint action, and this
is exactly the hardest problem (how separately-authored scripts settle
atomically), so the crux remains deferred; "mutually-distrusting" — no
hyphen after an -ly adverb.

---

## Site mechanics

- `zola build` (0.19.2) passes; 8 pages, no orphans, internal `@/` links
  resolve. Templates are clean and accessible (dark mode, semantic HTML).
- **`shy-heap.md` is untracked** — the deploy workflow builds on push to
  `master`, so it goes live the moment you commit and push it. Just know
  it's the one post that hasn't been through your commit-review ritual.
- Dating: four posts share the evening of 2026-07-16 with `+08:00` times to
  order them — works, and the commit history matches.
- Front matter is consistent; descriptions are good; the Atom feed is on.
- Minor: `config.toml` description says "breaking the occasional cipher" —
  with one cipher post out of eight, that's fine, but the blog's actual
  centre of gravity is "machines that review" (mode checker, Verus, Lean,
  POC, the compiler-as-reviewer theme in five of eight posts). The
  `borrow-checker-review` post even names the rungs. A tagline like
  "…and letting machines review my code" would describe the blog you
  actually have. Taste, your call.

## Cross-cutting observations

- **The recurring weakness is staleness, not error.** Three of the eight
  posts contain a claim that was true when drafted and false at publication
  because the code moved (89 choices, byte-for-byte, Docker script). Your
  repos save every benchmark and error message as artefacts — consider a
  pre-publish checklist that re-runs the greps for quoted numbers.
- **The two posts without links are the two that need them most** (they
  describe live, contested upstream work). The Miden post shows how much
  trust links buy: every quote there is checkable, and it shows.
- **The "reviewer ladder" is the blog's spine.** choice-tapes →
  mode-checker-review → borrow-checker-review → shy-heap's Lean ending is a
  genuine arc (compiler < effects discipline < proof assistant), and the
  posts cross-link correctly. The Mozak series is a second spine; fine, but
  the two haven't met yet — the day you write "Verus for the script
  settlement logic" or "the shy heap's matroid in Lean vs Verus", they will.
- **Name consistency:** upstream bcachefs PRs carry "Matthias Schorer
  <matthias@schorer.dev>" sign-offs under commits authored by "Matthias
  Goergens". If that's intentional, no action; if a reader (or Kent) goes
  looking, it will raise a question the blog doesn't answer.

## Prioritized fix list

1. `borrow-checker-review.md`: fix or hedge the `__GFP_IO`/NOFS paragraph
   (SWP_FS_OPS carve-out). This is the one error an expert reader will
   write to you about.
2. `shy-heap.md`: replace the Erickson citation with CLRS §16.5 (or drop
   it); fix the "conflicting definitions" sentence.
3. `frozen-ls.md`: remove `mapping_set_gfp_mask`; move dm-delay to the ktest
   phase; de-conflate GFP_NOFS from the SRCU fix; add links; credit Komzpa.
4. `choice-tapes.md`: 89 → 113; "byte-for-byte" → the README's phrasing.
5. `proof-of-0-equals-1.md`: correct the RPO mechanism sentence; pin or
   date-stamp the Docker claim.
6. `solution-in-search-of-a-problem.md`: drop or qualify token conservation;
   "trusted executor" → "single all-seeing executor".
7. `two-time-pad.md`: 17 GB → ~15 GB; "couple of dozen" → 35; the
   perfect-secrecy phrasing.
8. Everything in the per-post nits lists, at leisure.
