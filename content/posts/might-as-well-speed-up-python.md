+++
title = "Might as well speed up Python"
date = 2026-09-29T09:50:00+08:00
description = "To try out 250 dollars of free Claude Code cloud credits, I picked a project that needs only public data and asked Claude to make CPython faster. It found that Clang 19 slows the interpreter down by 8-11%, which I have reported upstream. Along the way it also turned up a flaky test and a side effect on the regex engine whose size depended heavily on code layout."
+++

Anthropic recently gave me 250 dollars of free credit for Claude Code's
cloud sessions, and I wanted to try them out. A cloud session only has
what you give it, so I picked a project that could work entirely off
public data, one I would otherwise have done on my desktop. I typed
this into my phone:

> I want you to take the latest cpython upstream main (or master or so)
> and look for making it faster. [...] As a rule of thumb, if you can get
> a 1% speedup across a general benchmark of cpython performance, that's
> already a win.

Most of the work was done by Claude; I steered, mostly from my phone,
and I checked what went out under my name.

## Measuring on borrowed hardware

The cloud container had four cores and was busy with its own builds most
of the time. Two runs of the *same binary* differed by up to 15%. Noise
alone just means doing more runs, but each run was also slow, and to
find 1% under that much noise we would have needed far more runs than
the box could manage. So I suggested using the CI of my
public fork of CPython instead: rip out the normal workflows and replace
them with our own measurements on throwaway branches.

GitHub's runners are noisy too, so the comparisons had to be designed
to cope with noise:

- Every job builds all variants ("arms") of an experiment on the same
  runner, then runs each benchmark with all arms back to back as fresh
  processes, in a freshly shuffled order, several rounds over. Slow drift
  on the machine hits all arms equally and cancels in the comparison.
- One arm is always the same binary as the baseline under another name.
  Any difference it shows is chance, so it shows how big a difference
  noise alone produces. It is also a canary: a difference bigger than
  chance would allow means the setup is biased.
- Builds with profile-guided optimisation (PGO: the compiler first runs a
  training workload, then optimises for what it saw) are not reproducible, and two
  builds of the same commit differ by up to about 1%. So the confidence
  intervals come from many independent builds, not from running one
  build many times.

The same-binary controls came out within about ±0.15%. They cannot show
layout effects, though, because the same binary has the same code
layout. Builds without PGO come out identical every time, so every
comparison between them rested on one draw from the linker's layout
lottery. PGO builds got a different layout each time, though not by
design. As the regex section below shows, layout alone can move a
benchmark by several percent.

## An intermission: everything else we tried

The dispatch find came early on the first morning, but measuring it
properly took most of the day, and a lot else happened around it.

Much of it was back and forth about method. I asked for a prior-art
search that covered diagnoses and plain complaints as well as proposed
solutions, and for a look at what made JavaScript engines fast. When
Claude proposed cheap proxies such as instruction counts, I said
micro-benchmarks only interested me as predictors of the general suite.
The randomised blocks, the interleaving of arms within a benchmark and the many independent builds
came out of those exchanges; so did an ablation of `-O3` against `-O2`,
after I remembered Don Stewart using evolutionary search to tune GHC's
flags.

One strategy worked well enough that I made it a rule: look for
performance *disputes* in CPython's history, where someone claimed a
slowdown and someone else called it noise. The arguments are usually
written up already; what is often missing is a measurement.

- The per-type method cache ([PR #150160][pr150160]) was merged with a 1%
  slowdown dismissed as noise. It is real: +0.50% [+0.25, +0.75] over ten
  pairs of builds, +6% on [richards][richards] (one of the
  pyperformance benchmarks: a simulation of an operating-system task
  scheduler, originally written in BCPL, heavy on attribute lookups and
  method calls). The cause is that type attribute lookups went from an
  inlined global cache (about 11 instructions) to an out-of-line call
  (about 45). Claude's attempt to inline the new lookup did not recover
  it.
- A claimed 0.9% slowdown from marking specialisation slow paths
  `noinline` did not reproduce (−0.12%, not significant).
- Doubling or quadrupling the GC's first-generation threshold makes the
  suite 2.2% or 3.8% faster. During the debate about the incremental GC,
  someone asked whether simply raising the existing thresholds would give
  the same gains, and nobody had measured it. But nearly all of it is the
  async_tree benchmarks; without them it is 0.3% or 0.6%, and we did not
  measure the memory cost.
- Turning off frame pointers saves about 1.2%, a bit below the 1.5-2%
  cost given in the PEP that turned them on.
- `-O2` is 5.4% slower than `-O3`, and leaving out -O3's passes one at a
  time shows that almost all of that is the larger inlining limits.

Claude's own optimisation ideas fared worse. A fast path in `_Py_Dealloc`
executed fewer instructions and was *slower* on every build. A fast path
for calling Python functions from C looked like a 0.37% win on six builds
and was nothing at twenty. Six builds were not enough to tell.

Along the way Claude also got [Stabilizer][stabilizer] working with
CPython. Stabilizer is a research tool I have been resurrecting that
re-randomises a program's code layout in every process, so that a change
which only helps by luck of alignment shows up as noise instead of as a
win. It also found and fixed a bug in Stabilizer: every function copy
was placed at 16 mod 32, so the interpreter loop landed on only a
handful of page offsets.

It made mistakes I had to catch. At one point it disabled address-space
layout randomisation in its instruction-count measurements to make them
reproducible. That is the opposite of what Stabilizer is for: it
randomises the layout on purpose, so that results average over layouts
instead of depending on one. When I pointed that out it agreed, and said
it had also pinned the hash seed everywhere.

## The find: one dispatch jump instead of 270

The dispatch story started as an item on a to-do list. Early on I had
asked Claude to look at compiler flags, even at changes to LLVM, and at
what made JavaScript engines fast. The research agent it sent out came
back with ten experiments, and the third was a dispatch-site audit.

CPython's interpreter loop uses computed gotos: each bytecode handler
ends with its own copy of the indirect jump to the next one, so the CPU's
branch predictor can learn patterns per opcode. Compilers like to merge
identical code, and in 2025 Nelson Elhage filed [gh-129987][gh129987]
because they were merging those copies. That issue was closed after
changes that only affected GCC. The agent's list also mentioned LLVM 19's
version of the problem, which is what had inflated the speedup first
reported for Python 3.14's tail-calling interpreter, but said it had
been fixed in LLVM 20.

So Claude wrote a small script to count the indirect jumps in
`_PyEval_EvalFrameDefault`, and found that GCC 13 produces 234 of them
for 232 targets (a few handlers have more than one exit): no merging, so
apparently nothing to do. I asked whether that was good or bad, and
whether we should find out how to control it. I remembered that the
first 3.14 tail-call numbers had been inflated by a compiler problem, and
guessed GCC might have one of its own. Then Claude compiled just
`Python/ceval.c` at `-O3` with a range of compilers:

| compiler | dispatch jumps |
|---|---|
| GCC 12, 13, 14 | 257 |
| Clang 18 | 269 |
| Clang 19.1.7 | **1** |
| Clang 21 | 268 |
| Clang 19 with `-mllvm -tail-dup-pred-size=1000` | 269 |

Clang merges all the dispatch jumps into one block and relies on a later
pass, tail duplication, to copy it back into every handler. LLVM 19 added
a limit to that pass ([llvm/llvm-project#78582][llvm78582]) to fix
compile-time blow-ups on huge switch statements, and CPython's loop is
far over the limit. LLVM exempted computed gotos from the limit in
20.1.1, but 19.1.7 is still around: it is FreeBSD's base compiler and
OpenBSD's, and Apple clang in Xcode 16.3 and 16.4 is affected in full,
Xcode 26.0 to 26.3 in part.

LLVM has fixed the bug, but Clang 19 is still in use, and we found no
report to any of the projects that build Python with it. Claude downloaded the published binaries and read the compiler string in each: FreeBSD's
python311 to python314 packages all say `Clang 19.1.7`, and so do
OpenBSD's. MacPorts builds on macOS 15 with Xcode 16.4, which has the
same problem.

Restoring the jumps is worth a lot. On GitHub's Linux runners, with
several independent builds per variant, on the pyperformance suite (the
standard CPython benchmark set; the geometric mean over its benchmarks,
with a 95% confidence interval):

| configuration | with the flag, vs without |
|---|---|
| PGO + LTO, Clang 19 | −8.4% [−9.5, −6.9] |
| thin LTO, no PGO (what FreeBSD ships), Clang 19 | −8.7% [−9.2, −8.3] |
| macOS arm64, Xcode 16.4, PGO + LTO | −11.4% [−13.2, −9.8] |
| macOS arm64, Xcode 26.3, PGO + LTO | −1.4% [−2.2, −0.7] |

Negative is faster. LTO is link-time optimisation, where the compiler
optimises the whole program at once when linking. That became
[issue gh-158283][issue] and
[PR gh-158286][pr], a configure check that adds the flag for the affected
compilers.

The first version of the patch did nothing in LTO builds, and Claude's
own control experiment caught it. Under LTO the machine code is generated
at link time, and the clang driver does not pass `-mllvm` options on to
the linker; it only warns that the argument is unused. Every build
already logged its dispatch-jump count, so a patched build with one jump
stood out. The fix passes the option to the linker plugin directly
(`-Wl,-plugin-opt=` for GNU ld and lld, `-Wl,-mllvm,` for Apple's ld64).

## Why I moved to my desktop

By the evening the cloud session had used up 125 dollars of the credit and
filed the issue and the PR, which I approved from my phone. It could not
attach python/cpython itself, because my fork is also called `cpython`
and the environment checks repositories out by name, so it filed them
from separate sessions it spawned for the purpose.

But the cloud box did not have my setup. It did not have my preferences:
it put an AI-generated footer on the issue and PR and co-author trailers
on every commit, which I had to have removed. It could not use my tools,
like the checker I run over prose for the tics language models fall into.
And it was slow. So I had it write a handoff note and continued on my
desktop, where Claude Code has my configuration, a 13th-generation i9,
and my laptops within reach.

## A flaky test

The first thing the local session looked at was a red CI job on the PR:
`test_threading`'s `test_set_and_clear` had timed out under the thread
sanitiser. That job was built with Clang 21, where the PR changes
nothing, so it was a flake. Before fixing anything I wanted a
reproducer, so Claude built the same configuration and ran the test on two cores shared with
four busy loops. It failed 9 times in 40.

The test starts five threads that call `event.wait()`, sleeps 50 ms in
the hope that they have all started waiting, and then calls
`event.set()` and `event.clear()`. On a loaded machine one thread only
arrives after `clear()` and waits five minutes. The fix waits until all
five threads are actually registered as waiters, and it failed 0 times
in 40 under the same load: [gh-158336][flake-issue] and
[PR gh-158337][flake-pr].

## The regex engine

I then had a subagent audit the issue and PR for claims without data
behind them. It found something I had missed: under LTO the flag applies
to the whole program, not only to the interpreter loop, and the regex engine
(`Modules/_sre/sre_lib.h`) also uses computed gotos. Its three matching
functions go from 7, 14 and 18 indirect jumps to 33 each. The one
benchmark that consistently got slower in the original runs was
regex_effbot, by 6-9%.

Split by the runners' CPU models, which the harness now records, the
slowdown was worst on AMD's Zen 3 (EPYC 7763). On my i9, my M4 Max and
my M3 MacBook Air, the PR made all three regex benchmarks *faster*. My
first idea was to use different code for different CPUs, which is
unworkable: nobody can test every CPU a Python binary will run on.

Claude also found a way to keep the regex engine merged without touching
the interpreter: make the shared dispatch block the target of an empty
`asm goto`, which LLVM refuses to duplicate. It works, and it is slower
on every machine I have: 12.8% slower than main on the M3, where the PR
as it stands is 5.7% faster.

So the PR sped up regex on every machine I own and seemed to slow one
benchmark down only on one CPU family. Then I asked whether the runners
had just been unlucky. We reran main and
the PR on GitHub with each build linked three times in a different
function order (`-Wl,--shuffle-sections`). On the EPYC 7763 runners
regex_effbot came out between 1% and 9% slower with the PR, depending on
the layout; two of main's own layouts differed by 4.7%. The three regex
benchmarks together showed no significant change. Under Stabilizer the
merged version was slower on Zen 3 as well. So the size of the
regex_effbot slowdown depended heavily on layout, and I left the PR as it
is. [The details are in the
PR description][pr], and the data is on a [branch of my
fork][data].

## What I took away

The dispatch find came from asking whether 234 jumps was good or bad, and
then counting the same thing with other compilers. None of the
hand-written optimisations the model tried produced a measurable win.

Code layout moved the results more than I expected: the regex_effbot
slowdown ranged from 1% to 9% across three link orders. For changes this
small I now want layout-randomised builds, with Stabilizer or at least a
few shuffled link orders.

Claude did the builds, the scripts, the statistics and a lot of reading,
much faster than I could have. I asked the questions, caught the
methodological slips, and decided what went out under my name. It took
most of three days.

[llvm78582]: https://github.com/llvm/llvm-project/pull/78582
[issue]: https://github.com/python/cpython/issues/158283
[pr]: https://github.com/python/cpython/pull/158286
[data]: https://github.com/matthiasgoergens/cpython/tree/clang19-dispatch-data
[pr150160]: https://github.com/python/cpython/pull/150160
[stabilizer]: https://github.com/matthiasgoergens/stabilizer
[richards]: https://pyperformance.readthedocs.io/benchmarks.html#richards
[flake-issue]: https://github.com/python/cpython/issues/158336
[flake-pr]: https://github.com/python/cpython/pull/158337
[gh129987]: https://github.com/python/cpython/issues/129987
