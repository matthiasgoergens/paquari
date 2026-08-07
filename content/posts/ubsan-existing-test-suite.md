+++
title = "Point UBSan at your existing test suite"
date = 2026-08-07T23:47:13+08:00
description = "Mature C projects run some sanitisers over some of their build configurations. I spent a few weeks of paternity leave hunting the gaps in that coverage matrix — CPython, glibc, the Linux kernel — and this is what fell out: the bugs, the noise ratio, and a trap I had laid for myself years earlier."
+++

I have a bit of an obsession with undefined behaviour in C, and this year,
some time on my hands due to paternity leave. So I spent a stretch of it
doing something that required much less cleverness than it should:
**building mature C codebases with sanitisers turned on, and running the
test suites they already have.**

That's the whole method. No fuzzer, no formal methods, no new tests.
Mature projects have spent decades accumulating suites that exercise
their edge cases, and the big ones do run sanitisers — some of the
sanitisers, over some of their build configurations. I just looked for
the gaps in that coverage matrix: the check nobody enables, the build
mode never paired with the sanitiser, the suite that has never met the
instrument at all. Every gap is a corpus of edge-case tests that some
sanitiser has never seen.

## How big is the gap?

Bigger than I expected, in projects that care a great deal about
correctness.

Take CPython. It runs UBSan in CI — but only in one configuration. The CI
matrix pairs UBSan exclusively with the default GIL build, so
`Python/uniqueid.c`, which only compiles in the free-threaded build, had
never been compiled under UBSan at all. The JIT has never been paired with
any sanitiser in CI either. And of CPython's 1,588 buildbots, the number
running UBSan is zero. GitHub Actions' single job is the project's entire
UBSan coverage, and it covers one build mode.

There's also a file where a project writes down that it stopped looking.
CPython's is `Tools/ubsan/suppressions.txt`: a list of files whose UBSan
reports are silenced so CI stays green. Suppression files are the best
starting point I know of, because every entry is a decision someone made —
often under time pressure — and never revisited. The same pattern shows up
wherever a project records its own defeats — xfail lists, skipped checks,
and, as we'll see in the kernel, a regression test that asserts a bug's
output as the expected value.

## What fell out of CPython

The suppressions file had four entries when I started; it's down to one
(which I could never reproduce and suspect is stale). Behind the others:

**ctypes bit fields** ([#154912](https://github.com/python/cpython/pull/154912)). The accessor macros shift negative
values and overflow signed shifts — eleven distinct UB diagnostics from
the existing `test_ctypes` alone. The fix does all bit manipulation in
`uint64_t` and sign-extends with the branchless `(v ^ sign_bit) -
sign_bit` idiom — which Mark Dickinson taught me in 2022, reviewing my
first attempt at this same bug. That PR sat for four years, and its fix
was incomplete: it wouldn't have cleared its own suppression. I closed it
and started over. Verification this time: an exhaustive differential
harness comparing old and new macros across every fixed-width type, bit
width, offset, storage pattern and value — 1,167,600 cases, zero
mismatches, under both gcc and clang.

**StringIO overseek** ([#154913](https://github.com/python/cpython/pull/154913)). `read()` after seeking far past the
end computes `buf + pos` even though it's about to return an empty
string. `buf` is `Py_UCS4 *`, so the byte offset is `pos * 4`, which
wraps for `pos = 2**62 - 1` and forms a pointer four bytes *below* the
buffer. `io.StringIO("abc").seek(2**62 - 1)` then `.read()` is the whole
reproducer, and the existing `test_io` suite hits the path the moment the
suppression comes off.

**Free-threaded refcount merging** ([#154915](https://github.com/python/cpython/pull/154915)) — the one that best makes
the CI-matrix point. The per-thread refcount delta — in practice almost
always −1 — is shifted left into the shared refcount word. Left-shifting
a negative value is undefined. Build the free-threaded interpreter under
UBSan and the very first line of output from `./python -c pass` is this
report; it fires again on every GC cycle. Across the full test suite:
1,762 UB reports, which broke 529 tests in 61 files — mostly tests
asserting that a subprocess printed nothing to stderr, which is its own
small lesson about how sanitiser output interacts with test suites. After
the one-line fix: zero, zero, zero. This bug was never obscure. It was in
a file CI structurally could not see.

The follow-up is a CI proposal ([#154927](https://github.com/python/cpython/issues/154927)): add UBSan × free-threading
and UBSan × JIT to the matrix. The strongest argument for the JIT row is
that it's already clean — 51,459 tests across 493 files in under five
minutes — so the row is a cheap regression guard, and this exact
hand-built combination is how gh-139269 (an unaligned store in the JIT
that segfaulted release builds) was found in the first place. The cost
objection doesn't survive measurement either: the heaviest configuration
I could construct, PGO + ThinLTO + UBSan, builds in 9m43s.

One more CPython bug belongs here, found by a different method: chasing a
flake. An intermittent CI failure landed on one of my PRs — as far as I
could reconstruct, infrastructure trouble; the failing test had nothing
to do with the change — and I went off to prove the red job was
unrelated. Hitting that particular flake was luck; following it was not.
Flake-hunting is a habit I've kept up at every job, partly because people
hate flakes with a passion and are disproportionately grateful to whoever
finally kills one, and partly because reliable reproduction —
historically the expensive half of any flake hunt — has become cheap. An
AI agent can usually grind out a reproducer on its own: rerun under load,
bisect the conditions, tighten the loop, all the busywork that used to
make chasing a flake a bad trade against feature work. It needs a bit of
guidance about the project's build and test setup the first time, and
that guidance is reusable for every later flake in the same project. And
once a flake reproduces reliably, it is just a normal bug. Proving this one unrelated meant reading the test
harness closely, and the harness turned out to contain a bug that had
been invisible for eight and a half years
([#155027](https://github.com/python/cpython/issues/155027)):
`test_asyncio`'s socket harness calls `self.fail()` from a worker thread,
and an assertion raised in `Thread.run()` never reaches the test runner —
so server-side failures have reported `ok` since 2017, identically across
every release from 3.6 to 3.14. The three-line fix promptly turned a
Windows CI job red with a real `ConnectionResetError` that had been
silently swallowed all along. A test suite can only be a sanitiser corpus
if its failures can actually reach you.

## glibc: layout decided by an undefined comparison

Not a sanitiser find, but the same undefined-behaviour family, and my
favourite bug of the batch ([BZ #32119](https://sourceware.org/bugzilla/show_bug.cgi?id=32119)). On 32-bit targets, glibc's
time64 `struct stat` helper tested `__BYTE_ORDER` without including the
header that defines it. In strict pre-POSIX-2008 feature-test modes both
macros are undefined, both evaluate to zero, and they compare equal — so
the code selects big-endian timestamp member order on little-endian
targets. The memory layout of a public structure, decided by a comparison
between two macros that don't exist. A second patch in the series kept
two accidentally-inherited reserved words from changing
`sizeof(struct stat)` with the feature macros. Adhemerval Zanella took
the series up the same day, squashing it together with his own fix for
two more architectures onto his staging branch (ecf95727a787), with a
backport planned.

## The kernel: where the test encodes the bug

The kernel work used a sibling method — archaeology rather than
sanitisers: find bugs somebody already diagnosed, sometimes already
fixed, that stalled for a reason written down somewhere.

The ext4 patch started from a syzbot report — a task hung in `do_rmdir` —
and turned out to be a livelock: on a corrupted filesystem, a retry loop
in `ext4_xattr_block_set()` re-selects the same unusable xattr cache
entry forever, while holding the directory's lock, so concurrent `rmdir`
callers block behind a spinning task. The fix makes the entry ineligible
before retrying. Measured under syzbot's own reproducer in QEMU: the
stock kernel hung in 6 of 8 trials; the patched kernel was clean in 20 of
20. Reviewed-by from Jan Kara. An ocfs2 fix from the same batch — a
cached cluster count updated in the wrong order during suballocator
reclaim — is in the -mm tree.

The best story, though, is `FIDEDUPERANGE`. The ioctl reports how many
bytes it deduplicated; since an unrelated 2018 refactor, it reports the
*requested* length even when the filesystem shortened the range. A
correct one-line fix landed in 2022 — and was reverted the next day,
because an xfstest, generic/517, asserts the buggy figure as its expected
output. A regression test encoding the bug it exists to catch is the
suppressions-file pattern wearing a different hat: the project wrote down
that it stopped looking, and the record then actively defended the bug.
My series pairs the kernel fix with the fstests fix, plus a survey
showing real consumers (deduplicators like rmlint) under-deduplicate and
report success because of it.

**AFL++**, as a coda. It entered this story as a tool, not a target — I
was using it to fuzz other things. When a custom-mutator setup misbehaved
under me, I took the tool's misbehaviour as a lead in its own right and
rebuilt AFL++ itself under ASan: a heap use-after-free in
`write_to_testcase()` — a mutator that returns its own buffer can have
the allocation reallocated out from under it. Fixed upstream, with two
neighbouring issues along for the ride. Same instinct as the flake: when
the instrument glitches, that's not noise in your experiment — it's a
bug report from a codebase you happen to have open.

## The noise, measured

A post that only lists wins would be marketing, so here is the other
column of the ledger.

Not every sanitiser check is worth turning on, and the differences are
large. For CPython: `local-bounds` ran clean over the full suite with
zero false positives — that one is free, turn it on.
`implicit-conversion` produced 167,202 diagnostics across 503 sites and
broke 598 tests: unusable without a triage effort nobody will fund.
`float-divide-by-zero` found exactly one site, in vendored expat, which
divides by zero deliberately, documents why, and is fine. And
`nullability` came back with zero findings — not because the code is
clean, but because CPython contains no `_Nonnull` annotations anywhere,
so the check had nothing to check. A clean run from an instrument that
cannot fire is not a clean bill of health.

Two consequences follow. For a maintainer, adoption isn't all-or-nothing:
a check too expensive or too noisy to gate every commit can still run on
a schedule — nightly, weekly — where its findings arrive as leads rather
than blocked merges. And for a bug hunter, the noisy end of the table is
not a write-off either: a check that produces a few hundred distinct
sites hands you a pre-sorted list of places to go digging. CI gating and
hunting are different consumers of the same output, with very different
tolerance for noise.

Which brings me to the trap I actually fell into. CPython builds with
`-fno-strict-overflow`, and that flag *silently disables* clang's
`pointer-overflow` sanitiser check — while gcc accepts the flag and never
implemented the check at all. I spent most of an afternoon producing
confidently wrong readings before noticing. The rule that came out of it,
which I now apply before trusting any clean result: prove the instrument
can see a deliberately planted bug first.

And there's an irony worth owning: that configure default is my own
doing. In 2022 I started an effort
([gh-96821](https://github.com/python/cpython/issues/96821), still open)
to move CPython off `-fwrapv` and towards `-fstrict-overflow` — measured
at the time as worth about 1% on pyperformance — and its merged first
step ([gh-96823](https://github.com/python/cpython/pull/96823), 2023)
added the `--with-strict-overflow` option whose default produces exactly
the `-fno-strict-overflow` that ate my afternoon three years later. The
migration stalled halfway, and the trap I fell into was one I had laid
myself.

And one more honest data point: the test I retracted. A reviewer
reasonably asked for a regression test on the StringIO fix. I wrote one,
then discovered it passes on unfixed builds too: with the compilers we
have today, this particular UB happens to have no observable effect —
and that is a fact about today's compilers, not about the code. Undefined
behaviour is a time bomb: the standard permits anything, and an ordinary
test can only assert what the current compiler happens to do with it. So
I removed the test, and argued the real regression guard is the
suppression-file entry staying deleted while UBSan is a hard CI gate —
the sanitiser sees the bomb without waiting for a compiler upgrade to set
it off. Sometimes the honest test is no test, and saying so in the review
thread beats shipping a test that asserts nothing.

## Tooling

I should say plainly how the work divides. I used AI tools to uncover the
undefined behaviour and to build the reproducers: driving the matrix of
build configurations, running the suites, aggregating thousands of
sanitiser reports into distinct sites, and turning a report into a
minimal, runnable trigger. The discovery instrument is the sanitiser and
the existing test suite; the agents are how one person points that
instrument at several codebases in a few weeks. The actual fixes are,
usually, easy to write by hand — most of the patches above are a handful
of lines, and the hard part was never the diff but the verification
around it. Every "usually" in this post has the same honest explanation:
I deliberately picked low-hanging fruit. Whenever something proved hard —
a fix that wanted a design discussion, a reproduction that wouldn't
stabilise — I moved on to the next candidate. The yield looks the way it
does partly because the hard residue got left behind, and that residue is
still out there for someone with more patience or more context. Even
going after C undefined behaviour in the first place was partly a
consequence of that policy rather than the other way round: I went where
the fruit hung lowest, and UB really is as easy to harvest as this post
makes it look. What genuinely surprised me is how much low-hanging fruit
there was, and is: I'm not done. Every finding was confirmed by a concrete reproducer on
unpatched upstream code before anything was sent, and the analysis errors
were mine. Communities are working out their norms on this; LLVM, for
instance, has a disclosure policy. The rule I'd defend anywhere: claims
backed by runnable evidence are checkable regardless of who or what
drafted them.

## The recipe

If you maintain, or contribute to, a C codebase:

1. Build with `-fsanitize=undefined -fno-sanitize-recover=all` and run
   your existing suite. (Those spellings are gcc's and clang's; other
   compilers have their own equivalents. If the sanitiser runtime won't
   link in your environment, trap mode — `-fsanitize-trap=all`, or
   `-fsanitize-undefined-trap-on-error` on older gcc — needs no runtime
   at all: violations become traps, and your test runner reports them as
   crashes.) A compiler other than the one you ship with is fine — better
   than fine: the sanitisers implement overlapping but different checks,
   as the pointer-overflow story above shows, so a second compiler is
   extra coverage. You've already paid for these tests; this is the flag
   that collects.
2. Find where the project wrote down that it stopped looking —
   suppression files, xfail lists, skipped checks, regression tests with
   suspicious expected values — and audit the entries.
3. Diff your CI matrix against what you actually ship. The bugs live in
   the pairings nobody builds.
4. Treat flakes as leads, not noise. Reproduction is busywork an agent
   can usually handle once you've taught it the project's build and test
   setup — teaching that pays for itself across every later flake — and
   once a flake reproduces reliably it is just a normal bug, often one
   that has been quietly corrupting your test signal for years.
5. Before reporting, get a positive trigger — a crash, a sanitiser
   abort, a measured wrong answer on unpatched code — and check your
   instrument can see a planted bug. Both directions of that rule saved
   me from myself in just these few weeks.

The cheapest bug you'll ever fix is one your existing tests already
catch, under a flag you're not passing.
