+++
title = "Exact running quantiles in linear expected time, if the input is shuffled"
date = 2026-07-19
description = "Maintain the exact 95th percentile of a stream, online, with no sketches and no approximation. Random arrival order turns a boring Θ(n log n) problem into a Θ(n) one: a Θ̃(√n) window if you may fail once in a blue moon, and — the punchline — the naive two-heap method already runs in expected linear time when you build it from pairing heaps."
+++

Everyone who runs a service has the dashboard: p50, p95, p99 latency.
Almost everyone computes those percentiles with a sketch — t-digest,
DDSketch, Greenwald–Khanna, KLL — trading exactness for tiny memory.
This post is about refusing that trade. You want the *exact* 95th
percentile of a stream, recomputed after every element, in the
comparison model. And you know one helpful thing: the elements arrive
in uniformly random order, every permutation equally likely. What is
the best asymptotic time and space?

The adversarial-order answer is boring. Keep everything in a balanced
search tree with order-statistic queries: Θ(n) space, Θ(n log n) total
time. The space part is not improvable either — one-pass exact
selection needs linear storage, a classical result of [Munro and
Paterson](https://www.sciencedirect.com/science/article/pii/0304397580900614)
(1980). So the interesting question is what the random order buys.

## The number that runs the whole problem: √n

At time m, the answer is the element of rank ≈ 0.05m from the top.
Watch the boundary between "the answer" and "just below the answer" as
elements arrive: it performs a random walk with constant variance per
step, and over n arrivals such a walk wanders Θ(√n) — the ballot
theorem scale, the law of the iterated logarithm, the same √n you see
in the DKW inequality for empirical distributions. Everything in this
post follows from that one scale.

Suppose first that you tolerate a failure probability of n^(−c) for a
constant c of your choosing. Then you only need to remember a *window*:
the elements within Θ(√(n log n)) ranks of the current answer, plus an
exact counter of how many discarded elements sit above the window. The
answer is exactly determined whenever it lies in the window, and —
important — you always know when it doesn't; the algorithm fails loudly,
never silently. The window slides and, relative to the stream, shrinks as the quantile
concentrates. Expected space Θ̃(√n), expected total time Θ(n): two
comparisons per element for almost everything, since only a Θ̃(1/√n)
fraction of the stream ever lands inside the window.

And Ω(√n) is necessary, even if you only cared about the *final*
answer: each of the Θ(√n) elements straddling the boundary is the exact
answer with probability Θ(1/√n), so nearly all of them must be stored.
[Guha and McGregor](https://doi.org/10.1007/978-3-540-73420-8_61)
(ICALP 2007) proved the formal version: one pass over a random-order
stream needs n^{1/2−o(1)} space to pin down the median even
approximately-tightly.

## Certainty costs linear space

Now the requirement that occupied me: always correct, on every
permutation — randomness may help the *expected* runtime, but never the
answer. Then no element can ever be certified as dead. Take any element
x you have seen, at any rank: x's rank from the top is monotone
non-decreasing, the target rank 0.05m is monotone non-decreasing, and a
suitable continuation of the stream — a flood of large elements, or
none at all — steers the two to coincide. Even the current maximum can
be forced to become the 95th percentile later. So an always-correct
one-pass algorithm must retain everything: Θ(n) space, end of story.
Randomness cannot buy space. It can only buy time.

And it can buy time: keep the Θ̃(√n) window for speed, but instead of
discarding the rest, throw it into two unsorted piles (above and below
the window, appended in O(1)). If the answer ever escapes the window —
the rare unlucky excursion — reselect it from scratch over the full
retained stream in Θ(n), a panic rebuild, and carry on. With margins
Θ(√(n log n)) the escape probability is n^(−c), so the expected rebuild
work over the whole stream is O(1). Total expected time Θ(n), i.e.
amortised constant per element, always exact. Add a safety valve — if
rebuild work ever exceeds a Θ(n) budget, switch permanently to the
boring balanced tree — and the worst case is capped at the classical
Θ(n log n) baseline at no expected cost.

That was where I would have stopped. It turns out the naive algorithm
already does better.

## The punchline: two pairing heaps

The textbook two-heap method maintains a max-heap of the bottom 95% and
a min-heap of the top 5%; the answer sits at the root of the big heap,
and sizes are rebalanced as the target grows. It is always exact, it
uses Θ(n) space, and its textbook cost is Θ(n log n). The claim: build
the two heaps as [pairing
heaps](https://www.cs.cmu.edu/~sleator/papers/pairing-heaps.pdf), and on
a shuffled input the expected total drops to Θ(n) — the same optimum as
the window machinery, in five lines of code.

The analysis is three facts, and the third is the one that needs the
random order:

1. **Pairing-heap insert is a single link, O(1) actual cost.** No
   bubbling. Almost every arrival becomes a fresh child of a root.
2. **Delete-root costs Θ(root degree), and the two-pass consolidation
   halves the degree, deterministically.** If the deleted root had d
   children, the pairing pass links them pairwise and the accumulating
   pass combines the ⌈d/2⌉ results; the surviving root ends up with at
   most ⌈d/2⌉ + 1 children.
3. **Only Θ(1) expected children accumulate between consecutive
   deletions of the same heap.** A rebalance fires when the arrival's
   landing side and the target size disagree — probability ≈
   2·0.95·0.05 ≈ 0.095 per arrival, so there are Θ(n) root-deletions
   overall (this is why binary heaps are hopeless: their delete is
   Θ(log n) even in expectation). But deletions hit each heap every
   ≈ 21 arrivals on average, so only ≈ 20 fresh children pile up
   between two deletions of the big heap, ≈ 1 for the small one.

The degree at deletion times therefore follows d′ ≤ d/2 + O(1) in
expectation — the noise is the geometric inter-deletion gap, bounded
whatever the current degree, which is all a contraction needs — so
E[degree] = O(1), so each deletion costs O(1) in expectation. Summing: n inserts at O(1)
actual, Θ(n) deletions at O(1) expected, queries free. Expected total
Θ(n), optimal because every element needs at least one comparison.

The heap choice is load-bearing in both directions. Binary heaps delete
in Θ(log n) expected time (sift-down runs to the leaves). Fibonacci
heaps amortise Θ(log n) per delete unconditionally — their proven
guarantee is input-independent, and no known analysis lets the random
order help. And a naive
"root with an unordered child list, rescan everything on delete"
structure pays the same Θ(d) per deletion but never contracts the
degree: the degree just grows by ~20 between deletions until it is
Θ(n), and the total is Θ(n²). The two-pass tournament inside the
pairing heap is precisely the degree-halving mechanism that keeps the
recurrence stable. Self-adjustment is doing real work here.

## It degrades gracefully, which the window does not

Nothing in the two-heap *algorithm* uses randomness — only the analysis
does. The size invariant is maintained on any arrival order, so if your
"random" input turns out to be adversarial, the answers stay exact and
the cost slides smoothly to the classical Θ(n log n) worst case (the
original Fredman–Sleator–Tarjan amortised bounds), with no regime
detection and no safety valve. Actual cost is Θ(root degree at delete
times), which simply grows as the input's interleaving departs from
random — a gentle slope, not a cliff (how gentle, exactly, is delicate:
whether this restricted workload can even realise the known adversarial
near-Θ(log n) sequences is not obvious). My window-plus-rebuild scheme
achieves the same cap only through its explicit fallback, which is a
clunky thing to need.

One caution, because pairing heaps have earned it before: [Fredman
proved](https://dl.acm.org/doi/10.1145/320211.320214) (JACM 1999) that
they do *not* support decrease-key in O(1) amortised time, contrary to
the standing conjecture. General adaptivity claims about pairing heaps
are delicate, which is exactly why the argument above is a direct
expected-actual-cost analysis of this specific workload rather than an
appeal to some general theorem about their niceness.

## Coda

The literature around this is tidy: [Munro and
Paterson](https://www.sciencedirect.com/science/article/pii/0304397580900614)
started it; Guha and McGregor mapped out random-order quantiles
([PODS 2006](https://doi.org/10.1145/1142351.1142390), [ICALP
2007](https://doi.org/10.1007/978-3-540-73420-8_61), [SICOMP
2009](https://doi.org/10.1137/07069328X)) including the one-pass Ω(√n)
lower bound and exact selection with polylog space in O(log log n)
passes; [Chakrabarti, Jayram and
Pătraşcu](https://www.cs.dartmouth.edu/~ac/Pubs/soda08-median-ds.pdf)
(SODA 2008) matched the pass count from below, together resolving Munro
and Paterson's open question. A fuller write-up with all
the derivations lives in a [companion
repo](https://github.com/matthiasgoergens/running-95th-percentile).

And a familial note: pairing heaps keep showing up on this blog. The
soft heap inside the [shy heap](@/posts/shy-heap.md) is built on them
too — two quite different problems where the same self-adjusting
structure does something the textbook heaps cannot. There is probably
a third post in that pattern.
