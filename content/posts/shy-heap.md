+++
title = "The shy heap: priority queues in linear time, if you promise not to peek"
date = 2026-07-18
description = "Run n insert/pop-min operations and ask only who survived: soft heaps plus matroid duality answer in O(n), slipping past the sorting bound that makes heaps cost log n. An idea I have carried for years, with a Rust implementation, a stalled paper, and a Lean proof now taking shape."
+++

Comparison-based priority queues cost logarithmic time per operation,
and that is not an engineering shortfall, it is a theorem. Feed n items into any exact
heap and pop them all: the pops come out sorted, so the operations
together must pay the Ω(n log n) that comparison sorting costs. If you
want to go faster, you must give something up. The famous way to do
that is Chazelle's [soft heap](https://en.wikipedia.org/wiki/Soft_heap),
which gives up *correctness*: it runs in amortised constant time per
operation (for a fixed error rate) and is allowed to corrupt a bounded
fraction of its keys in exchange.

This post is about giving up something else. Keep correctness; give up
the *order of results*. You hand over the whole sequence of operations
up front — insert this, pop the minimum, insert that — and you ask
only one question: at the end, which items had been popped, and which
survived? Not when anything was popped, not in what order, only the
final partition. I call the data structure that answers this a **shy
heap**: it will tell you whether an item ever left, but it is too shy
to say who left when. Under that restriction, the whole sequence can
be processed in O(n).

The restriction is not cosmetic; it is load-bearing. If you also
learned the order of the pops, you would have sorted, and the lower
bound would apply. If you even learned which pop removed which item,
you could sort. The shy heap sits exactly on the edge of what the
lower bound permits, which is some evidence that it is the right
weakening rather than an arbitrary one.

## Where the question came from

The problem has been rattling around my notes for years, wearing a
scheduling costume. Alice has n items of homework; each takes a day,
each has a deadline and a penalty for missing it. Which items should
she do, and which should she abandon, to minimise the total penalty?
Sort by deadline (a bucket sort, since no deadline beyond day n
matters), walk the days backwards, keep a max-heap of available items,
do the most valuable one each day. That greedy is correct — it is the textbook
task-scheduling matroid ([CLRS](https://en.wikipedia.org/wiki/Introduction_to_Algorithms)
§16.5) — and the heap makes it O(n log n).

But notice what the answer actually needs. A schedule can be
reconstructed in linear time from just the *set* of items that get
done. The heap is computing a full ordering and throwing it away. That
smelled, by analogy with
[median-of-medians](https://en.wikipedia.org/wiki/Median_of_medians)
and the [expected linear-time MST
algorithm](https://en.wikipedia.org/wiki/Expected_linear_time_MST_algorithm),
like a problem that wanted a linear-time answer. It took me an
embarrassingly long time to find the two ideas it wanted.

## Idea one: soft heaps, wearing a cleaner API

A soft heap corrupts keys, and every classical treatment drags the
corruption machinery through every statement. My favourite
side-product of this project is an API in which corruption is never
mentioned at all, building on the simplification of [Kaplan, Kozma,
Zamir and Zwick](https://arxiv.org/abs/1802.07041). Let `extract-min`
return a *set* of items rather than a single item, and demand just two
things of a run over any operation sequence S, comparing against what
an exact heap would have popped:

1. every item the exact heap would have popped is in the soft heap's
   pops (no false survivors), and
2. the excess — soft pops that the exact heap would have kept — is at
   most ε·(number of inserts).

Corruption becomes an implementation detail behind a two-line
contract. The payoff of the contract's first line: any item the soft
heap has *not* popped by the end is a certified survivor of the exact
run. And the soft heap delivers this in amortised constant time per
operation for fixed ε.

So run the sequence once through a soft heap with ε = 1/3. At least
r − εm of the true survivors are certified (r survivors among m
inserts), and they can be set aside for good. Repeat on what is left.
This works beautifully — until it doesn't. Each pass shrinks the
inserts towards the number of pops and then stalls: on a sequence that
pops nearly everything, almost nothing gets certified, and the
recursion treads water. My draft paper, I will admit, currently ends
at almost exactly this point, one section before the punchline.

## Idea two: every heap trace has a dual

Here is the punchline. A trace of inserts and pop-*mins* has a
complementary trace of inserts and pop-*maxes*, computable in linear
time without a single key comparison, whose exact run leaves behind
precisely the items the original run pops. In the scheduling costume
this is obvious once said aloud: walk the days backwards and each pop
schedules the best item still available; walk the days forwards and
each pop discards the cheapest item that no longer fits before its
deadline. Every item is popped in exactly one of the two walks, so
each walk's survivors are the other's pops.

Run the soft heap over *both* traces. The primal pass certifies true
survivors: at least r − εm of them. The dual pass certifies true
victims: at least k − εm, where k is the number of pops. Between them,
all but 2εm items get a verdict in one direction or the other — the
awkward dependence on k has cancelled out. With ε = 1/3, each round
settles a third of the remaining items and hands the unresolved
two-thirds to the next round: a geometric series, total work three
times one two-pass round. That is the whole algorithm.

The two passes are not an ad-hoc trick, and this is where it gets
pretty. Fix a trace — the inserts and deletes in a fixed order, each delete
free to remove whatever it likes — and consider all the sets of items
you could be left holding. The possible outcomes form the bases of a
[matroid](https://en.wikipedia.org/wiki/Matroid) — I call these *heap
matroids*, reluctantly coining a name because the literature's
near-synonyms ([nested, laminar,
Schubert](https://www.math.lsu.edu/~oxley/LaminarMatroids_revised.pdf))
almost — but not quite — coincide. Running a min-heap is finding the
maximum-weight basis. The family is closed under [matroid
duality](https://en.wikipedia.org/wiki/Dual_matroid), and the
primal/dual pass conversion above *is* the duality map. The soft heap
identifies part of the optimal basis; duality lets it identify part of
the complement; the recursion inherits the rest. Selection — all
inserts before all pops — falls out as a special case, so this is
incidentally yet another linear-time selection algorithm, and the
maximum-weight balanced-parentheses subsequence problem succumbs to the
same reduction.

## Where this stands

A [Rust implementation](https://github.com/matthiasgoergens/shy-heap-rs)
exists and is public, together with a new soft-heap implementation
based on pairing heaps — the first of a family, neat in practice,
tricky to analyse, and a story for another post. The paper draft stops where I
said it does. And the part I am currently excited about: a Lean 4
formalisation of the whole argument, with the soft heap as an
observable black box, the heap-matroid structure proven and bridged to a
maximum-weight-basis predicate over Mathlib's Matroid library, and an
end-to-end theorem —
the recursive two-pass algorithm returns exactly the survivor set of
the exact heap — now standing, with the linear-time recurrence
formalised alongside it. Getting a proof assistant to review this
argument has already reshaped it once, and that experience is the next
post in the reviewer-ladder series this blog keeps returning to.

The shy heap has spent years as a private obsession. Writing it up is
how private obsessions either become results or get honestly retired,
and I intend to find out which this one is.
