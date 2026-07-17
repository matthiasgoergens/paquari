+++
title = "What the borrow checker won't review"
date = 2026-07-16
description = "Two bcachefs bugs that cost me a week each. The borrow checker would have caught neither: one is an allocation in the wrong context, the other a loop that never ends, but a proof assistant catches the second. Different reviewers for different bugs."
+++

A recurring theme on this blog is handing your code to a compiler and
letting it find the bug. I still believe in that. But I spent last week
on bcachefs (swap files on a copy-on-write filesystem, and the hardening
that keeps them from deadlocking) and I want to be honest about where
"let the compiler review it" runs out.

It's a live question for the project, not a rhetorical one. bcachefs is
in the middle of a slow rewrite into Rust, and it already carries a set
of Verus proofs for its trickier data structures (the eytzinger search
tree, the `bpos` ordering). So "which of these bugs does the machine
catch" is something the codebase is actively betting on.

Here is the borrow checker's slice, stated plainly: in-memory ownership
and data races. Use-after-free, double-free, a `&mut` that aliases, two
threads racing on the same word. That's a real and valuable slice, and a
Rust port of the extent-refcounting code really would close it. But for
a filesystem it's a *small* slice, and the two bugs that actually ate my
week live above it.

## The allocation in the wrong context

Swap on a copy-on-write filesystem means the kernel writes swap pages
*through the filesystem*. So the memory-reclaim path (the code the kernel
runs when it's low on memory) can call back into the very filesystem that
is trying to free memory. Every filesystem knows this shape and guards
against it: allocate with `GFP_NOFS`, or run under a `PF_MEMALLOC_NOFS`
scope, and reclaim won't re-enter the filesystem.

Swap doesn't respect that guard. Swap writeout from reclaim is gated by
`__GFP_IO`, not `__GFP_FS`, so `PF_MEMALLOC_NOFS` stops reclaim from
re-entering the filesystem the ordinary way but does nothing to stop
reclaim issuing a *swap write*, which re-enters the same filesystem
anyway. Put swap on bcachefs and the whole `NOFS` contract quietly
changes meaning. This deadlocks:

```
reconcile thread
  -> btree_write_buffer_flush     [holds the write-buffer lock]
  -> darray resize, kvmalloc(GFP_KERNEL)
  -> direct reclaim -> swap_writeout -> bch2_swap_rw   [wants this same fs]
  -> bio_alloc, mempool empty, sleeps forever
journal write: blocked on the write-buffer lock
```

A `GFP_KERNEL` allocation, held under a lock the journal needs, on a
filesystem that happens to host swap. The fix is one word: `GFP_NOIO`
instead of `GFP_KERNEL`, which forbids reclaim from starting I/O and so
cannot recurse into swap. The hard part isn't the fix, it's *finding
every site*: every allocation that runs under a lock the write or journal
path can wait on, across the allocation.

Now: which type would have flagged that? None of them. There is no type
in Rust, or C, or OCaml, for "this allocation happens in a context where
these locks are held and reclaim can recurse into this subsystem." It
isn't a property of a value; it's a property of the dynamic execution
context you're standing in. The borrow checker tracks ownership and
aliasing; it does not track *effects*. A `GFP_NOIO` allocation and a
`GFP_KERNEL` allocation have exactly the same type.

You could *build* a type for it. Thread a capability (call it a `NoIo`
token) through every function that runs under those locks, and make the
allocator demand the token as an argument. That's an effect discipline,
and Rust's type system is expressive enough to carry it where C's isn't.
But it's something you architect deliberately, function by function; it's
not a review the compiler performs for free. Out of the box, this entire
class of bug is invisible.

## The loop that never ends

The dedup background pass (reconcile) walks extents and shares identical
ones. When two extents collide in the dedup index but a byte-for-byte
check says their contents actually differ, it's supposed to give up on
that extent. It didn't: it left the extent's "pending" flag set, so the
next reconcile pass picked it up again, checked it again, gave up again,
forever. Under the right input the filesystem's journal simply freezes,
live-locked on an extent it will never resolve.

This is a liveness bug. The code isn't doing something *wrong*, it's
failing to make *progress*. And progress is exactly where static analysis
famously stops: you cannot decide, in general, whether an arbitrary loop
terminates. The borrow checker won't catch it. Neither will exhaustive
matching or any "make illegal states unrepresentable" trick, because the
illegal state here isn't a value you can rule out of the type, it's time.

Here's the turn, and it's the one that belongs on this blog. The tool
that *does* reach this bug is the proof assistant. Verus (the same one
bcachefs is already using to prove its search trees) supports `decreases`
clauses: you name a measure that must strictly decrease on every
iteration, and it proves the loop terminates. Reconcile as a fixpoint
with a well-founded "work remaining" measure is precisely a `decreases`
obligation. Written that way, the bug isn't a live-lock in production;
it's a proof you cannot discharge, at compile time.

## Three rungs

So two bugs, two reviewers, and neither is the borrow checker:

- **In-memory ownership and data races**: the borrow checker. Real, free,
  and the smallest slice of a filesystem's bugs.
- **Allocation context / effects**: an effect discipline you build by
  hand. Rust can carry it; nothing hands it to you.
- **Liveness and termination**: a proof assistant with a `decreases`
  clause. The one rung tall enough to see the loop that never ends.

"Let the compiler review your code" is still the right instinct: I
wouldn't be writing a shrink engine around a mode checker otherwise. But
the free reviewer reviews the least, and the bugs that cost the most days
were one and two rungs above it.

Which is why I find bcachefs's Rust-*and*-Verus direction more
interesting than a plain rewrite. Safe Rust would close the memory-safety
class outright. But the project is also proving things with Verus, and
that is the rung that reaches the liveness bug the borrow checker never
could. The reviewers stack. You just have to know which one you're
asking, and for a filesystem the answer is rarely the cheap one.
