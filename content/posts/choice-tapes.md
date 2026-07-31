+++
title = "Your generators already know how to shrink"
date = 2026-07-16T21:48:00+08:00
description = "A Conjecture-style choice-tape shrink engine for base_quickcheck, built on a seam OCaml gets almost for free."
+++

Here is a line from base_quickcheck that surprised me:

```ocaml
(* shrinker.ml *)
let int = atomic
```

`atomic` means "never produce any shrink candidate". The same goes for
`int32`, `int64`, `float`, `char`, and `bool`. When your property
fails on `766135`, base_quickcheck reports `766135`. Only structure
shrinks: lists drop elements, but the elements themselves stay
whatever they were.

I do not think this is a bug. I think it is a principled surrender.
A `Shrinker.t` is a function from a value to smaller candidate values,
and it cannot know what the generator that produced the value would
have been willing to produce. Shrink an even number by halving and you
may hand an odd number to a test that assumed evenness; shrink a field
of a struct and you may violate an invariant the generator carefully
established. Jane Street chose the safe corner of a bad trade-off:
no shrinking rather than invalid shrinking.

The rest of the OCaml ecosystem took the other well-known exit.
QCheck2 and Bam both use integrated shrinking in the Hedgehog style:
a generator produces a lazy rose tree of values, and combinators
transform whole trees, so shrinking composes through `map` and
`filter` for free. It is a real improvement, and it still has a known
soft spot: monadic bind. When the outer value shrinks, the inner tree
was built from a value that no longer exists, and the usual answers
(regenerate, or freeze the outer value) both lose.

There is a third model, and as far as I can tell nobody had brought it
to OCaml: Hypothesis's [Conjecture
engine](https://hypothesis.works/articles/how-hypothesis-works/).
Record every random decision the generator makes as a typed choice. To
shrink, edit the recorded tape and run the generator again against it,
accepting the edit only if the test still fails and the new recording
is shorter or simpler. The property that makes this special: a shrink
proposal cannot violate a generator invariant, because the proposal is
not a value, it is an input to the generator. Whatever comes out went
through every filter, every smart constructor, every dependent bind,
exactly like the original.

I spent last week porting this model into [Rust's
proptest](https://github.com/proptest-rs/proptest/pull/658), which
required migrating strategies one by one to record typed choices. Then
I looked at base_quickcheck and realised something pleasing: OCaml
gets this almost for free.

## Splittable_random is a perfect seam

Every base_quickcheck generator draws its randomness from one
sequential `Splittable_random.t`. And look at the primitives:

```ocaml
val bool : t -> bool
val int : t -> lo:int -> hi:int -> int
val float : t -> lo:float -> hi:float -> float
```

Every draw arrives with its bounds attached. In proptest I had to
migrate each strategy so the engine would know the type and range of
each decision; here the seam is typed already. Wrap a handful of
functions so they record to (and replay from) a tape when one is
installed, and every existing generator participates, including
everything `[@@deriving quickcheck]` produces. I verified the claim on
a derived record type: one generation records 113 typed choices, and
replaying that tape under a completely different RNG seed reproduces
the identical value. The vendored base_quickcheck in my repo is
unmodified except for its dune file and one portability fix; a dune
workspace resolves the `splittable_random` library name to my shim, and
that is the entire integration.

(An earlier draft of this post said that `Generator.fn`'s randomly
generated functions do not shrink, because `fn` splits the random
state and split-off streams were not taped. That is no longer true:
the tape now keys sub-streams by the split identity and the
argument's hash, so generated functions shrink too, down to
boundary-exact observed behaviour, and the reported counterexample's
function stays callable after the run. That deserves, and will get,
its own post.)

## Does it work?

Six properties, 100 seeds each, identical failing examples handed to
both shrinkers. "Stock" is base_quickcheck's own greedy loop, exactly
as `Test.run` performs it. These figures are a dated snapshot of the
original six benchmark rows: tapecheck
[`24fb1b3`](https://github.com/matthiasgoergens/tapecheck/commit/24fb1b3a5640b941317db93004da2f6c4d939a25),
31 July 2026, built with stock OCaml 5.3.0 and Dune 3.24.0. The benchmark
has since acquired two additional adversarial cases; they are not folded
into this original six-row comparison.

| property | stock minimal | tape minimal | stock avg calls | tape avg calls |
|---|---|---|---|---|
| int uniform, fail iff >= 123457 | 0/100 (worst `766135`) | 100/100 | 0 | 38 |
| pair, fail iff a + b >= 100 | 0/100 (worst `(481 781)`) | 100/100 | 0 | 22 |
| list, fail iff length >= 3 | 0/100 (worst `(12 100 61)`) | 100/100 | 5 | 178 |
| list, fail iff sum >= 100 | 0/100 (worst `(15 91)`) | 100/100 | 4 | 173 |
| filtered evens, fail iff >= 100 | 0/100 (worst `21150`) | 100/100 | 0 | 90 |
| bind: length-prefixed list, sum >= 100 | 0/100 (a 64-element monster) | 100/100 | 0 | 56 |

One honest caveat lives in the stock column: its call counts are near
zero, because it has almost nothing to try. The tape engine buys its
minimality with test executions.

The bind row is the point. `let%bind len = ... in list_with_length
~length:len ...` has no derivable shrinker at all, so stock reports
whatever it generated. The tape engine returns `[100]`, the global
minimum, in 49 test executions on average: it lowers the length choice
while deleting the choices of one element, replays, and the generator
rebuilds a consistent shorter list every time.

My favourite single number is from a three-way chained bind, `a` in
10..1000, `b` in 10..a, `c` in 10..b, failing unconditionally. The
tape engine's first proposal sets every choice to its target; replay
walks the binds again, so the dependency structure holds by
construction, and it lands on `(10, 10, 10)` on its first proposal.

## What the tape sees inside base_quickcheck

Recording every draw gives you an X-ray of generator internals, and
two findings seem worth reporting upstream.

First, `Generator.int_inclusive` is a weighted union: 5% `return lo`,
5% `return hi`, 90% uniform. The constant branches record a tape of
one choice (the branch selector), and escaping to the shrinkable
uniform branch would lengthen the tape, which a shortlex-ordered
search must refuse. The `lo` branch is harmless — that case is already
minimal — but one failing case in twenty starts inside the `hi`-branch
trap.
Hypothesis structures the same boundary bias inside the sampler, one
typed choice with a biased distribution, and every case is
shrinkable. Distributional bias belongs in the draw, not in generator
structure. This is a small, concrete change to suggest.

Second, one list element is not one choice. `list_generic` draws a
length, a size-budget distribution, a permutation, then elements, so
removing an element means deleting a shuffle draw and a value draw
together. My deletion pass learned to remove small contiguous blocks
alongside lowering the length. This is worth knowing for anyone
building a coverage-guided fuzzer on top of these generators, too.

## The OxCaml part

The engine builds and runs under OxCaml (the 5.2.0+ox overlay, Base
and ppxlib at v0.18 preview) with a handful of compatibility shims in
my vendored copies, none in the engine. A determinism check — 200
find-and-shrink cycles of the bind-heavy property above — gives
byte-identical results on both compilers, down to the same 10129
total shrink attempts. For a testing tool, determinism across
compiler forks is a feature worth stating.

Shrink attempts are also embarrassingly parallel: independent replays
of edited tapes, racing to find an accepted improvement. My first
engine had exactly one piece of state in the way, a convenient global
the shim consulted, and OxCaml's mode checker rejected it by
inference, with a paper trail, before any parallelism existed to go
wrong. The refactor it forced bought a 4.6x wall-clock win the same
afternoon (and Flambda2 runs the sequential engine 12 percent faster
while we are at it). That story, with the compiler's actual review
comments, is [the next post](@/posts/mode-checker-review.md).

## Where this could go

The [repo](https://github.com/matthiasgoergens/tapecheck) has a
drop-in `Tape_test` module mirroring
`Base_quickcheck.Test.run/run_exn/result`; existing suites switch by
renaming one module, and the `quickcheck_shrinker` they already
declare is simply ignored. The honest upstream path is small: tape
hooks in splittable_random's dozen primitives, behind a no-op default
([proposed](https://github.com/janestreet/splittable_random/pull/2)).
And a typed tape is a better fuzzing corpus than raw bytes, so there
is an AFL++ custom-mutator story here too, but that is a later post.

If you maintain OCaml property tests and have a failing case that
reported a 64-element monster where a one-element list would do, I
would love to hear whether this engine finds it.
