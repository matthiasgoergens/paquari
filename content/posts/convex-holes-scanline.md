+++
title = "Scanning for convex holes"
date = 2026-08-17
draft = true
description = "I revived my 2010 Haskell solution to Project Euler 252 to find out why it was slow. It was slow because it was wrong, wrong because of one word, and slower than the standard approach for a reason worth measuring: its search states genuinely cannot be merged. Plus a Lean port, and a machine-checked proof of the lemma I kept getting almost wrong."
+++

In 2010 I solved [Project Euler problem 252](https://projecteuler.net/problem=252)
— find the largest *hole* in a cloud of 500 pseudo-random points, a
convex polygon with vertices on the points and no point strictly
inside — and left behind a Haskell repo whose README apologises: "My
solution isn't very clever anyway." Sixteen years later I dusted it
off with two questions. Why does it still build only against a 2010
GHC? And why was it so slow, when other people's solutions ran in
seconds — was my *idea* slow, or my implementation?

The modernisation was routine (Stack, `lts-24.55`, a couple of renamed
imports, one dead symlink). The speed question turned out to be the
wrong question.

## Wrong and slow travelled together

My commit history told the story before the profiler did. One commit
says "gives right answer for n=500". The next says "solved, but bugs
seem to remain". HEAD, rebuilt today, produces an answer **65% too
large** — it confidently reports a "hole" that contains other points.

The diff between those two commits changes many things at once (a
breadth-first frontier became a depth-first branch-and-bound), but the
regression is one word. The working version sorted points
lexicographically, x then y. The broken version sorts with a
comparator that compares **x only**. Points sharing an x coordinate —
plenty, with coordinates drawn from a range of 2000 — now arrive in
arbitrary order, and the sweep's emptiness invariant quietly assumes
they don't. Restoring `sort` fixed every size I could test, verified
three ways: the old breadth-first commit, the fixed depth-first
version, and an independent Python implementation of a completely
different algorithm all agree everywhere they were compared.

The part worth keeping is how the bug and the slowness fed each other.
The search is a branch-and-bound: it prunes any partial polygon whose
optimistic bound falls below the best complete hole seen so far. A
wrong "best" that is 65% too large sounds like it should prune *more*,
and at n=500 it did — the buggy version was fast there, for the wrong
reason. But at n=300 the bug produced a *differently* wrong best that
mis-steered the search into 32 seconds of work the fixed version does
in 1.7. If your branch-and-bound has inexplicable performance cliffs,
check whether it is also wrong. The two travel together, because the
same quantity — the incumbent best — drives both the answer and the
pruning.

## Anatomy of the scanline

My 2010 idea was a sweep. Move left to right across the points; carry
a set of *open* polygons, each a lower chain and an upper chain plus
two half-plane constraints (`inf`, `sup`) that pin down where the
polygon may still grow. When the sweep reaches a new point that falls
inside a polygon's cone, that polygon must decide: absorb the point
into the lower chain, absorb it into the upper chain, or commit to
excluding it — above or below — by tightening a constraint. Four
children per open polygon per interior point, and only the area bound
keeps that from being exponential.

Fixed, this runs in about 1.9 seconds for n=500 (after replacing the
per-step re-walk of both chains with an incrementally carried area,
worth 2.8×). The standard approach people use — I'll get to it — runs
in 0.37 seconds and carries a worst-case guarantee. So: is the
scanline *idea* slow?

## Two negative results, honestly measured

The first thing I noticed when trying to speed it up: the sweep never
looks *inside* the chains. Its future depends only on the chain heads,
the two constraints, and the accumulated area — a five-component
tuple. The chains are dead weight. That immediately suggests sharing: merge
search states that agree on the five-tuple, keeping the best area.
This is how dynamic programming is born.

Measured: merging removes **3.1%** of the states. Ninety-seven percent
of the 80 million surviving states are unique, because the constraints
remember history — every excluded point tightened a cone through that
particular point — and the two chains' constraints multiply.

Second attempt: don't require *equality*, remove *dominated* states.
There is a genuinely sound dominance relation here — same chain heads,
at least as much area, and a cone that contains the other's cone on
the half-strip ahead of the sweep. Because each constraint passes
through its chain head by construction, cone containment reduces to
two integer sign tests, and each bucket of states becomes a
three-dimensional Pareto frontier. I implemented it, verified it
still produces correct answers at every size, and measured it.

It removes 3.1%. The same 3.1%. Dominance bought essentially nothing
beyond deduplication, and the reason is the interesting part: a state
with a wider cone almost always paid for it with less accumulated
area. The trade-off is real, so the frontier states are genuinely
incomparable — each one carries information the others lack. No
merging discipline fixes that. The representation itself resists
sharing, and that, not constants, is why the scanline loses.

## The representation that shares

The approach that works — the standard one for this problem — anchors
the polygon at its lexicographically smallest vertex
and decomposes it into the fan of triangles from that anchor.
Emptiness of the polygon is then *per fan triangle*, because
consecutive fan vertices are consecutive polygon vertices, so each
fan chord is a polygon edge. A joint property of the whole polygon
becomes a local property of each edge, and the DP state collapses to
"the last edge": every chain history ending in the same edge merges,
by construction. That is the sharing the scanline could not have, and
it is why the same computation drops from 1.9 seconds to 0.37, with an
O(n³) guarantee in place of data-dependent pruning.

Two facts fall out free of charge, which I had not appreciated before
writing it down: convexity at the anchor *and* at the closing vertex
are automatic consequences of processing candidates in angular order.
The DP only needs to enforce left turns along the chain.

## The two traps, and a machine-checked apology

Twice in this exercise my informal geometric reasoning was almost
wrong in ways that would have survived testing on random inputs.

First: a point lying exactly on an internal fan diagonal — collinear
with the anchor and a chain vertex, but closer — is strictly inside
the polygon without being strictly inside *any* fan triangle. The
per-edge check misses it; you need a separate exclusion (no interior
chain vertex may have a closer point on its ray).

Second: when I formalised "convex polygon, counter-clockwise", my
first definition — all consecutive turns strictly left — admits
**pentagrams**, which wind twice and for which the fan decomposition
is simply false. The right definition is by supporting lines: every
vertex lies strictly left of every edge it doesn't belong to.

Both traps concern exactly one lemma: *a point is strictly inside a
convex polygon iff it is strictly inside some fan triangle or on some
open internal diagonal*. This is the statement the whole fast
algorithm leans on, and the statement I fumbled twice. So I stated it
in Lean 4 — integer points, cross products, nothing fancier — and
handed it to [Aristotle](https://harmonic.fun/), Harmonic's automated
theorem prover, as *prove or disprove*, with my proof sketch in the
docstring.

Fifty-six minutes later it returned a proof. I re-checked it locally
the paranoid way: the theorem statement is byte-identical to the one I
submitted, the project builds, there is no `sorry`, and
`#print axioms` reports only the three standard Lean axioms. The
proof follows the sketch — barycentric expansion of the cross product
for one direction, a least-index wedge argument for the other — and
the collinear case, the one I mistrusted most, gets its own honest
branch. The lemma now compiles, sorry-free, against current Mathlib.

## Coda: Lean as a systems language

Since the spec was in Lean anyway, I ported the fan DP itself —
mutable arrays inside `Id.run do`, `Int64` arithmetic — and compiled
it natively. Same answers at every size. The numbers for n=500:

| implementation | time |
|---|---|
| scanline, 2010, fixed (GHC) | 5.4 s |
| scanline, incremental area (GHC) | 1.9 s |
| scanline, shared / Pareto (GHC) | 50–60 s |
| fan DP (GHC, `-O2`) | 0.37 s |
| fan DP (Lean, `lake build`) | 0.59 s |

One footgun worth recording: my first Lean binary, compiled by hand
with `lean -c` and `leanc -O3`, ran the same workload in 1.18 s —
twice as slow as what `lake build` produces with no flags at all.
Benchmark the build system's output, not your hand-rolled pipeline.
Lean landing within 1.6× of GHC on array-crunching code, while the
same repo holds a machine-checked proof of the algorithm's key lemma,
is a combination I did not have in 2010.

The remaining gap, for a future post: proving the DP itself correct
against the spec — the fan lemma was the hard geometric core, but the
induction connecting `dp[i][j]` to "maximum hole" is still on paper.
