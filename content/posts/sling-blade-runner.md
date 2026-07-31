+++
title = "The heuristic that lied about the ceiling"
date = 2026-07-20
description = "I hand-built a solver for the Sling Blade Runner puzzle, got it stuck at 233, and concluded that multi-word overlaps barely helped. Then an off-the-shelf solver proved the real optimum was 310 — a third higher — in ten seconds. The heuristic hadn't found a hard limit; it had found a comfortable rut."
+++

*The [ITA Software hiring puzzles](https://web.archive.org/web/20080828230011/http://itasoftware.com/careers/hiringpuzzles.html), preserved on the Internet Archive — Sling Blade Runner is among them.*

Sling Blade Runner asks: how long a chain of overlapping movie titles can you
find? The end of one title has to overlap the start of the next — `Sling Blade`
+ `Blade Runner` — and each title is used once. You get a list of 6561 titles and
a note that heuristic answers are fine, "a reasonable tradeoff of efficiency and
optimality." That last clause turned out to be the whole story, just not the way
I expected.

## The graph, and a real bottleneck

Model each title as a directed edge from its first word to its last word. A chain
is then a walk through this graph, and its structure is worth looking at before
writing any code, because the structure decides everything.

Two facts jump out. First, it is *almost a DAG*: 6062 of the 6561 edges never lie
on a cycle, and nearly all the cyclic content is trapped in a single
strongly-connected component. Second, there is a brutal bottleneck. **1603 titles
start with `THE`; none end with it.** In the single-word model that means at most
one `THE…` title can ever appear in a chain — you can leave the word `THE` once
and never return. `MY`, `MR`, `HOW`: same story, pure sources.

So the longest chain is a long acyclic path with one detour through the cyclic
core. I got there the embarrassing way — first mis-modelling it as an Eulerian
problem (a valid chain of *four*), then as a pure longest path (91), before
combining a min-cost-flow-balanced core with exact longest-path arms on the
backbone. That gave **232 titles**, and it reads beautifully as one run-on
sentence: `THE GOSPEL OF JOHN Q AND A PERFECT MURDER AND MURDER IN THE FIRST BLOOD
DIAMOND MEN CRY BULLETS OVER BROADWAY…`.

## The part where I was wrong

The puzzle explicitly allows *multi-word* overlaps: `License to Kill` +
`To Kill a Mockingbird` = `License to Kill a Mockingbird`, joined on two words.
My solver ignored them, and `THE` is exactly where they should matter. Nothing
ends in the bare word `THE`, but plenty of titles end `…OF THE DEAD`, and
`THE DEAD ZONE` is right there waiting. I measured it: multi-word overlaps add 808
new join pairs and make 262 of those 1603 locked-out `THE`-titles reachable. Even
better, a title ending `…THE X` can feed *another* `THE`-title, so THE-land stops
being a one-shot sink and grows into a 620-title strongly-connected component.

I was sure this would unlock a lot. So I switched models — with multi-word
overlaps a title no longer has a fixed first/last interface, so titles become
*vertices* and a chain becomes a longest *simple path* — and wrote a local search:
restarted greedy, insertion, iterated local search, the usual kit. It climbed to
**233** and stopped. One title better than before.

I ran it harder. Bigger perturbations, thousands of restarts, a hundred and
seventy seconds of dedicated search on the core alone. It sat at 233. So I wrote
the honest-sounding paragraph: *reachability is not path length; the extra edges
are too sparse and scattered to lengthen the longest path; a nice case of an
intuition that measurement refutes.* It felt like wisdom. It was a local optimum
with good PR.

## The ten-second refutation

Longest simple path is NP-hard, but 6561 vertices is not large, and there is a
clean way to hand it to a solver: a **circuit constraint over a dummy node**. Give
every node a self-loop meaning "skipped"; connect a dummy node to and from every
real node to mark the path's start and end; force a single circuit; maximise the
number of non-skipped nodes. OR-tools CP-SAT has this constraint built in.

The whole graph, 6561 vertices, solved to **proven optimality in about ten
seconds: 310 titles.** Not 233. The upper bound met the solution, so it is not a
better heuristic — it is *the* answer, with a certificate that nothing longer
exists. Restricted to single-word overlaps only, the optimum is 235 (my 232 was
nearly there). So multi-word overlaps are worth **+75 titles, a third more** —
exactly the big win my instinct had promised and my local search had buried.

I cross-checked with HiGHS, formulating it as a prize-collecting single circuit
with MTZ subtour elimination. It agrees the core's longest path is 301 — but takes
around 150 seconds where CP-SAT's specialised circuit propagator takes 0.2, a
fair reminder that the right constraint beats a generic formulation of the same
problem.

## What I actually learned

The uncomfortable part is not that my heuristic was suboptimal. Heuristics are
supposed to be. It is that it was suboptimal *and confident*, and I wrote its
confidence down as a fact about the problem. A local optimum doesn't announce
itself as one; it feels exactly like a ceiling. My "multi-word overlaps barely
help" was not a measurement — it was the shape of the basin my particular search
happened to fall into, dressed up as a law.

The fix was not a cleverer heuristic. It was spending ten minutes to phrase the
problem in a form an exact solver already understands, and letting it both beat my
answer by 33% *and* prove the number. There is a general lesson I keep relearning,
including [the last time](@/posts/two-time-pad.md):
find the substructure that some standard tool solves exactly — an Euler circuit, a
Viterbi lattice, a min-cost flow, a circuit constraint — and reach for it before
trusting your own hand-rolled cleverness, especially when that cleverness starts
telling you the ceiling is lower than you hoped. It is often just telling you where
it got tired.

## Postscript: when the ceiling stops being reachable

The whole satisfaction above was the *certificate* — not just a long chain, but a
proof that nothing longer exists. So I couldn't resist asking what happens on a
bigger list. ITA's 6561 titles came from MovieLens. I used the MovieLens 25M
dataset; after normalising its 62,423 rows, 57,130 titles remained. Same CP-SAT
model, pointed at the bigger graph.

The chain grew to **4,087 titles** — thirteen times the 310 — and every overlap
verifies. But here is the honest part: it is *not* optimal, and I cannot tell you
how far from optimal it is. On the small list CP-SAT closed the gap in ten seconds
and handed me a bound that met the answer. On the big list the cyclic core alone is
19,255 titles, a longest simple path through it is genuinely hard, and the solver's
upper bound stayed thousands above whatever it had found. 4,087 is just the longest
chain it had produced when I stopped — and I stopped for unglamorous reasons: each
warm-started incumbent was 2910, 3439, 4012, then 4087. The first two gains were
comparable (+529 and +573); the final round added only 75, and the long runs kept
getting killed before they finished.

So this number is a *floor*, not an answer — the exact mirror image of the small
case. There the pleasing thing was a proven ceiling; here the honest thing is
admitting there isn't one within reach, only a chain that keeps lengthening for as
long as you feed it compute, with no way from the inside to know how much is left.
That is its own lesson about scale: the exact-answer move that felt like mastery at
6561 titles quietly stops paying out at 57,000, and the grown-up response is to say
so — to report 4,087 as a partial result with its gap wide open, not to quote it in
the same breath as the 310 as though it were the same kind of number.
