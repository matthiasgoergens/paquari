+++
title = "What the borrow checker won't review"
date = 2026-07-16T23:59:00+08:00
description = "Two bcachefs bugs that ate my week. The borrow checker would have caught neither: one is an allocation in the wrong context, the other a loop that never ends, but a proof assistant catches the second. Different reviewers for different bugs."
+++

A recurring theme on this blog is handing your code to a compiler and
letting it find the bug. I still believe in that. But I spent last week
on bcachefs (swap files on a copy-on-write filesystem, and the hardening
that keeps them from deadlocking) and I want to be honest about where
"let the compiler review it" runs out.

It's a live question for the project, not a rhetorical one. bcachefs is
in the middle of a slow rewrite into Rust, and it already carries a set
of [Verus](https://github.com/verus-lang/verus)
[proofs](https://github.com/koverstreet/bcachefs-tools/tree/master/verus-proofs)
for its trickier data structures (the eytzinger search
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

Swap once made that guard subtler because swap writeout from reclaim is
primarily gated by `__GFP_IO`, not `__GFP_FS`. But the mm has known how
to handle filesystem-backed swap since 2022: [NeilBrown's
d791ea676b66](https://github.com/torvalds/linux/commit/d791ea676b66)
("mm: reclaim mustn't enter FS for SWP_FS_OPS swap-space", v5.19) makes
reclaim refuse to enter the filesystem for a swap-cache folio under
`__GFP_IO` alone when the swap space goes through filesystem operations,
which is exactly how swap on bcachefs works. An explicit `GFP_NOFS`
allocation or an active `PF_MEMALLOC_NOFS` scope strips `__GFP_FS`, so
it does close this swap-writeout path.

The observed allocation had neither protection. Its unscoped
`GFP_KERNEL` retained both `__GFP_FS` and `__GFP_IO`, leaving reclaim
free to write a swap page straight back into the filesystem that was
trying to free memory. This deadlocks:

```
reconcile thread
  -> btree_write_buffer_flush     [holds the write-buffer lock]
  -> darray resize, kvmalloc(GFP_KERNEL)
  -> direct reclaim -> swap_writeout -> bch2_swap_rw   [wants this same fs]
  -> bio_alloc, mempool empty, sleeps forever
journal write: blocked on the write-buffer lock
```

An unscoped `GFP_KERNEL` allocation, held under a lock the journal
needs, on a filesystem that happens to host swap. The fix is one word: `GFP_NOIO`
instead of `GFP_KERNEL`, which forbids reclaim from starting I/O and so
cannot recurse into swap. `GFP_NOFS` would also have blocked this
particular recursion; the patch chose `NOIO` as the stronger, simpler
invariant for every allocation under the write-buffer locks: do not
start reclaim I/O there at all. It also upgraded an existing NOFS scope
to NOIO. That upgrade was not needed for the documented SWP_FS_OPS
recursion, but it makes the invariant uniform. The hard part isn't the
fix, it's *finding
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
forever. Hand it an extent that shares a checksum with different
contents, and the reconcile queue never drains: live-locked on work it
will never resolve.

This is a liveness bug. The code isn't doing something *wrong*, it's
failing to make *progress*. And progress is exactly where static analysis
famously stops: you cannot decide, in general, whether an arbitrary loop
terminates. The borrow checker won't catch it. Neither will exhaustive
matching or any "make illegal states unrepresentable" trick, because the
illegal state here isn't a value you can rule out of the type, it's time.

Here's the turn, and it's the one that belongs on this blog. The tool
that *can* reach this bug is the proof assistant, but not through a bare
termination check on the worker loop: each pass already terminates. The
useful obligation is on the state transition. Under quiescence, every
successful terminal dedup attempt must either convert or index the
extent, or clear its `dedup_pending` flag; only an explicitly transient
outcome may leave it pending for a retry. The buggy byte-mismatch path
returned success while preserving the flag, violating that postcondition.

That contract matches the repair. Six terminal cases — no usable
checksum, compressed data, size mismatch, a hit on the extent itself, a
source-checksum mismatch, and a byte mismatch — now pass through
`dedup_skip_terminal()`, which re-peeks the current key, clears the flag,
and commits. I/O errors, transaction restarts, and data changing while
the locks are dropped deliberately retain it. Once the local contract is
proved, a quiescent reconcile-to-fixpoint driver over a finite set of
pending extents really can use "work remaining" as a `decreases` measure.
The liveness proof needs both pieces; the bug would make the first one
fail at compile time.

## Three rungs

So two bugs, two reviewers, and neither is the borrow checker:

- **In-memory ownership and data races**: the borrow checker. Real, free,
  and the smallest slice of a filesystem's bugs.
- **Allocation context / effects**: an effect discipline you build by
  hand. Rust can carry it; nothing hands it to you.
- **Liveness and progress**: a proof assistant with state-transition
  postconditions and, under quiescence, a `decreases` measure. The one
  rung tall enough to see the work item that never retires.

"Let the compiler review your code" is still the right instinct: I
wouldn't have built [a shrink engine around a mode
checker](@/posts/mode-checker-review.md) otherwise. But
the free reviewer reviews the least, and the bugs that cost the most days
were one and two rungs above it.

Which is why I find bcachefs's Rust-*and*-Verus direction more
interesting than a plain rewrite. Safe Rust would close the memory-safety
class outright. But the project is also proving things with Verus, and
that is the rung that can express the progress contract this bug
violated and the conditional termination argument above it. The
reviewers stack. You just have to know which one you're asking, and for
a filesystem the answer is rarely the cheap one.
