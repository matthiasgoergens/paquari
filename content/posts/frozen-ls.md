+++
title = "How a frozen `ls` turned into swap on bcachefs"
date = 2026-07-16T23:58:00+08:00
description = "My tiered bcachefs desktop hard-locked under load. Chasing why an `ls` could freeze for a full minute took me from a stuck user to sending patches: an SRCU starvation fix, working swap files, and a stress harness that was the only reason I trusted any of it."
+++

I've run bcachefs as my desktop root filesystem for a couple of years
now. Five devices (one NVMe as an SSD tier, four spinning disks as the
cold tier) with data written to the fast tier and quietly migrated to
the HDDs with compression in the background. It is a genuinely nice
setup to live on: you get SSD latency for the working set and HDD
capacity for everything else, and mostly you forget it's there.

Mostly. This isn't my first bcachefs puzzle. A couple of years ago my
filesystem kept mounting at on-disk version 1.4 while my tools had moved
on to 1.7, insisting a downgrade was required, and no `set-option`
incantation would budge it; even a freshly created partition started at
1.4. I ended up reading the kernel's `bcachefs_format.h` to work out
why, and the answer was quietly educational: the on-disk version is
capped by your *kernel*, not your tools. My 6.8 kernel simply didn't
know about anything past 1.4; the userland tools run ahead, kicking the
tyres on new features before the kernel gets them. Kent was patient
about it on IRC. It was the first time I went into the bcachefs source
to answer my own question, and it would not be the last. What changed
this time was that the machine didn't merely confuse me: it hard-locked,
and I decided to find out why instead of rebooting.

## The hang

The setup was a heavy dev day: something like twenty-seven IDE sessions
open, tens of gigabytes of anonymous memory, and a fat pile of kernel
slab holding bcachefs metadata. Then the whole machine stopped
responding and I had to hold the power button.

The tell, once I started paying attention, was subtler than a full lock:
kick off a big background write (a `dd`, or just a large `sync`) and
basic foreground commands would freeze. `ls`, `stat`, `grep`: gone for
thirty, sixty, sometimes more seconds, then back as if nothing had
happened. If you've run a tiered bcachefs setup you may have felt this
as an occasional "why did the terminal just stop" during a big file copy
or delete. I later found other people describing exactly that in the
wild (upstream issues
[#934](https://github.com/koverstreet/bcachefs/issues/934) and
[#636](https://github.com/koverstreet/bcachefs/issues/636) are the two I
kept returning to), which was reassuring in the way that only "it's not
just me" can be.

The two obvious explanations were both wrong, and being wrong was the
useful part. It wasn't the OOM killer: it never fired. And swap didn't
save me either; if anything the machine was worse for having it. So the
memory pressure was real but it wasn't the *cause*. Something was holding
still while the disks caught up.

## What was actually holding

The culprit was an SRCU lock held far too long. SRCU (sleepable
read-copy-update) is what lets a reader hold a lightweight lock across a
sleep. bcachefs takes one per btree transaction. Under a heavy
writeback storm, the background workers pushing data down to the slow
HDD tier could monopolise the btree locks while holding their SRCU read
locks for as long as the physical writes to spinning rust took, and a
foreground metadata lookup (the thing `ls` does) would sit behind them.
That's your freeze: not a deadlock exactly, a starvation,
foreground work stuck behind background work stuck behind a slow disk.

The fix lived in the lock discipline: dropping the long-held locks
properly (`bch2_trans_unlock_long`) so a reader never sits on its SRCU
lock across a blocking wait. With it, `time ls -la` came back in under
ten milliseconds during aggressive background ingestion. The
terminal stopped freezing.

## Reaching out, and getting reviewed in public

I wrote the patches up and, being honest, posted them to
[r/bcachefs](https://www.reddit.com/r/bcachefs/)
partly to reach Kent, and partly out of impatience. Public interest for
a nifty improvement never hurts, but mostly I wanted eyes on it.

I got them, fast, and the review was instructive in a way I didn't
expect. Kent looked at it (and mentioned, in passing, that this was the
first run of *POC*, [his own AI assistant](https://poc.bcachefs.org),
doing a first pass of the code
review; we are all, it turns out, working with something in the loop
these days). His feedback landed on exactly the thing I'd gotten too
clever about: my `drop_locks_long_do()` helper was a nice idea but
insufficient, because `unlock_long()` is an *automatic transaction
restart*, and you don't want that firing implicitly. The right shape is
to unlock, block for a few seconds, and *then* unlock long. He was also
politely unconvinced that my mechanism fully explained the freezes, and
asked the only question that matters: what does your testing show?

And he pointed me at [ktest](https://github.com/koverstreet/ktest) (his
existing test harness) instead of the QEMU contraption I'd been building
to fake a slow disk with blkio write throttling. No need to roll your
own framework.

## What it turned into

That review is the hinge of the whole thing. "What does your testing
show" sent me to build the tests properly, inside ktest. I used
`dm-delay` to reproduce the slow-HDD reconcile contention, then built a
separate, calibrated swap-pressure suite: repeated rounds near full swap
utilisation, swapoff under pressure, no-swap controls, and ablations of
the proposed safeguards. The small [upstream smoke
test](https://github.com/koverstreet/ktest/pull/63) is only the basic
reclaim-pressure case by design, not a version of the later matrix that
was cut down for upstream; the [fuller stress
suite](https://github.com/koverstreet/ktest/pull/98) contains the tests
behind the swap analysis. Which is how I could then do the thing I
actually wanted: make **swap files work on bcachefs**.

Swap on a copy-on-write filesystem is a funny problem. The kernel used
to reject a swapfile on bcachefs outright, complaining about "holes,"
because it went through the generic `bmap()` path. bcachefs implements
the modern `SWP_FS_OPS` interface instead, so the filesystem itself
translates swap offsets to physical blocks through the btree at I/O
time and the hole-checks simply don't apply. (The first real bug was
sillier than any of that: my new module was being shadowed by the stock
`bcachefs.ko` that the initramfs loaded first. `mkinitcpio -P` and a
reboot.)

The deep part was making it stable under real memory pressure. Swap I/O
is itself invoked by reclaim, so it has to reach the storage path without
recursively entering reclaim and waiting on the filesystem resources it
needs to make progress. My historically tested branch was deliberately
armoured: `memalloc_noreclaim_save()` around swap I/O, bkey-buffer and
disk preallocation, pinned btree nodes, and, in an earlier version, a
pre-reserved btree cache. One correction matters here: `GFP_NOFS` is
already sufficient to stop reclaim writing an `SWP_FS_OPS` swap page
back into the same filesystem; a blanket `NOIO` requirement was an
overstatement.

The ablations also made the story narrower. Removing the `PF_MEMALLOC`
propagation alone reproduced the stalls and deadlocks; this test did not
show a measurable benefit from pinning or the cache reserve. The
[armoured discussion branch](https://github.com/koverstreet/bcachefs-tools/pull/787)
preserves that history and the remaining mechanisms, although its stress
results were measured before the latest rebase. The [current reduced
proposal](https://github.com/koverstreet/bcachefs-tools/pull/646), posted
by Darafei Praliaskouski (Komzpa), deliberately strips out the
swap-specific memory machinery and keeps `bch2_swap_rw()` as a thin
direct-I/O dispatch path. It has build validation, but the maximum-
exhaustion result belongs to the historical armoured implementation, not
to that reduced head. The sign-offs carry us both.

All of this happened next to a third thread: online filesystem
shrinking, a long-requested feature that another developer, jullanggit,
and I both swung at around the same time — his implementation was the
cleaner one. Different story, same neighbourhood.

## The thing I'd actually tell you

The technical fixes are the visible part, but the honest centre of this
is a line I wrote in that Reddit thread and mean completely: the stress
test setup was the only reason I could make any of these changes with any
confidence at all. I am not Kent; I have not internalised this codebase
the way he has. What let a user-who-got-annoyed turn into someone sending
patches to a filesystem was not cleverness; it was a reproducer that
turned "I think this is the bug" into "watch it deadlock, then watch it
not." A frozen `ls` is a bad bug report. A test that freezes on demand
is a place to stand.
