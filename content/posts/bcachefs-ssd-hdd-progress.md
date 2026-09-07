+++
title = "Towards full SSD performance in mixed SSD/HDD bcachefs"
date = 2026-09-05T08:58:00+08:00
draft = false
description = "Progress towards keeping SSD foreground work responsive while bcachefs uses HDDs for capacity in the background."
+++

[Bcachefs](https://bcachefs.org/) is a copy-on-write filesystem for Linux. One of its attractions is that a single filesystem can span a mixture of devices: fast NVMe SSDs, older SATA SSDs and large spinning hard disks, rather than requiring a matched set of drives.

Its [foreground and background targets](https://bcachefs.org/Caching/) let you use that mixture as a storage hierarchy. New writes can go to the SSDs, with data moved to the HDDs in the background. The SSD copies can stay around as a cache. In principle, that gives you SSD responsiveness with HDD capacity.

Alas, placing foreground data on SSDs does not yet reliably isolate it from the slow devices. Foreground writes can still be held up by filesystem-wide work that waits for the slowest member, dragging their latency down to HDD speeds—or worse when a drive stalls. The data need not be written to that HDD for it to delay the operation.

Slow writes are not the only symptom I have encountered. In [an earlier investigation](@/posts/frozen-ls.md), ordinary `ls`, `stat` and `grep` commands could stop for thirty to sixty seconds or more during heavy background writes. Later I also saw small-file reads stall and `syncfs` take far too long to return. These are not necessarily one bug: lock contention, readahead, writeback and durability dependencies can each turn background activity into a foreground pause.

This is the short overview of the work so far. The companion pieces cover [the filesystem and block-layer mechanisms](@/posts/bcachefs-slow-member-mechanisms.md) and [experimental bcachefs swap, failed boots and the testing method](@/posts/bcachefs-swap-and-testing.md) in more detail.

## The target

The priorities matter, because it is easy to spend a week optimising the wrong thing.

First, protect SSD foreground latency. I do not need a zero-percent regression. I can live with some lost throughput or a modest relative latency increase. I do not want an SSD operation to become a seconds-long wait because an unrelated HDD is busy.

Second, make background work genuinely useful without catastrophic failure modes. A test that keeps the foreground fast by preventing all movement is not a success.

Third, eventually use as much idle HDD capacity as possible. This is a worthwhile stretch goal, not a prerequisite for an initial improvement.

Fourth, later, I want a desktop mode that can complete interactive work at RAM speed, including a carefully bounded relaxation of `fsync`. That is a different durability contract, not permission for ordinary filesystem acknowledgements to lie.

## The organising model

Giving background I/O low priority is useful, but insufficient. The I/O scheduler can choose among requests it can see. It cannot prioritise a foreground request that has not yet obtained a request slot, pull a command back out of a drive's firmware, or know that a journal entry needed by an SSD writer is waiting for an unrelated HDD operation.

The most useful model has therefore been to look for synchronous dependency edges. Where can foreground progress inherit a wait on a slow member while holding a finite shared resource?

The journal—the recovery log which makes groups of filesystem updates crash-safe—is one example. It is finite and ordered. If an early entry cannot finish, later work eventually runs out of room behind it. A local slow-device dependency can then become a filesystem-wide pause. Request slots, write-buffer space, locks on filesystem indexes, superblock updates and memory-reclaim completion can create similar amplification.

That model has led to several distinct pieces of work:

- Limit the mover by outstanding work rather than imposing a permanently low bandwidth ceiling. The goal is to exploit idle capacity while leaving little enough already committed work that a new foreground request has a chance.
- Scope the cache flushes issued before a journal commit to devices that actually owe relevant durability, instead of letting a debt-free HDD delay an SSD-only commit.
- Keep the old SSD copy authoritative until a demoted HDD copy is durably committed, while batching expensive flush work rather than issuing Force Unit Access for every small extent.
- Stop optional readahead—speculative reads beyond the immediate request—from filling scarce request slots. A filesystem-wide cap is deployed; a selected-device and access-aware policy is still experimental.
- Keep maintenance paths from silently defeating placement policy.

The [mechanisms companion](@/posts/bcachefs-slow-member-mechanisms.md) explains the evidence and qualifications behind those bullets. Some are clean prospective fixes; others remain prototypes. A stack trace showing a journal wait still does not, by itself, identify the operation that caused the wait.

## CopyGC made placement part of the problem

CopyGC is bcachefs's copying garbage collector. Storage is allocated in buckets; when a partially used bucket should be reclaimed, CopyGC moves its remaining live extents elsewhere so the bucket can be reused.

That makes CopyGC a maintenance operation with the power to change physical placement. A dedicated stock-upstream VM reproducer showed it moving an SSD-resident file onto an HDD even with reconciliation disabled. The filesystem remained structurally valid while violating the performance policy I wanted.

The two-patch treatment makes CopyGC prefer the extent's source device and back off cleanly when that destination has no room. In the latest exact-composition VM, all 3,400 tested extents retained one replica on each SSD and none on the HDD while CopyGC evacuated 1,218 buckets and changed more than a thousand pointers on each SSD. Readback, shutdown and offline fsck passed.

The physical machine is already running a module that contains those patches, but CopyGC is still disabled pending a bounded physical activation. Reconciliation—the background engine that brings data into line with placement, replication and compression policy—is enabled and has moved useful data to the HDD tier.

## Readahead showed why favourable controls matter

Upstream bcachefs derived a filesystem-wide readahead window by summing a per-device allowance. That is useful when a sequential read really fans out across several devices. It is dangerous when one file has one relevant slow replica: optional reads can occupy request slots and make the demand read wait.

In a targeted VM shape, cutting a 10 MiB aggregate window to 128 KiB removed observed tag waits and reduced maximum completions from roughly 2.4 seconds to about 40 ms. But simply reverting the summed window is not a general answer. A striped positive control showed that the larger stock window can provide a real throughput benefit.

The physical setup now has a 2 MiB aggregate cap. The more interesting prototype admits readahead according to the selected device and observed access pattern, declining optional work instead of blocking for a tag. It retained high sequential throughput in the tested favourable case while greatly improving the hostile one. It still needs broader qualification.

## Swap is a stress workload, not a deployed feature here

Most Linux swap is a dedicated block partition or a simply mapped swapfile. The bcachefs work discussed in this project is different and experimental: the filesystem itself services swap I/O through bcachefs operations. It is not in the physical dogfood and is not something an ordinary bcachefs user should assume exists today.

I use it in VMs because swap traffic is an unusually effective way to expose liveness bugs. The kernel may need a swap write to finish before it can reclaim the memory that some completion path needs. That has exposed issues in reclaim context, request admission, wakeups and write-buffer reservation which ordinary file benchmarks missed.

[The swap and testing companion](@/posts/bcachefs-swap-and-testing.md) explains that lineage, the parts that held up under ablation, and the apparatus failures that repeatedly produced confident but wrong stories.

## What has improved

The newest physical dogfood is a stock Arch kernel with only bcachefs replaced by the exact experimental module. That keeps NVIDIA, input devices and unrelated kernel code close to the distribution setup. It completed the full production boot contract without the immediate corruption diagnostics that sank an earlier attempt.

Reconciliation is active, strict foreground and metadata placement point at the SSD group, and background placement points at the HDD group. A slow-member edge-inventory VM moved 384 MiB while the slow devices saw only allowlisted idle mover traffic. In an eight-boot paired VM run, active/control foreground throughput ratios were 0.917–0.990 and active p99 latencies stayed between 149 and 182 microseconds.

Those results are genuine progress, not proof of completion. On the physical filesystem, deliberately noisy runs overlapping package builds still produced seconds-scale foreground tails. Active versus paused reconciliation did not show a stable causal direction: some active cells were worse, but some paused cells were also very slow. That realistic failure is being kept rather than excluded as an inconvenient nuisance.

The current candidate also contains fixes or prototypes for scoped durability, strict placement, CopyGC placement, readahead admission, write-buffer ownership, reconciliation lifecycle and emergency shutdown. Several can become independent upstream submissions. Reliable reproducers are at least as important as the patches, particularly where the mechanism remains disputed.

## What remains

The immediate gap is not “make the benchmark a few percent faster”. It is explaining and removing the remaining human-visible physical tails while useful background work continues. The next experiments distinguish delay before submission from devices that hold a small number of queue slots until completion, and correlate foreground stalls with journal, preflush and per-device latency state.

Real hardware is less polite than a fixed-delay VM. One HDD is drive-managed SMR, and the machine has seen SATA command timeouts and link recovery. In one incident a 4 KiB read failed after roughly 34 seconds; bcachefs retried another replica and repaired the affected copy. A useful mixed-tier filesystem should contain that kind of badness whenever the data layout permits it.

There is also an unavoidable capacity edge. If the SSD tier is genuinely full and foreground allocation requires demotion to create space, foreground progress really does depend on background progress. That needs headroom and watermarks, not a scheduling trick.

The work is therefore in a much better place than “rate-limit the HDD and hope”, but it has not reached “adding HDDs is unnoticeable”. The aim remains the same ordinary one: let the slow disks be useful without making everything else wait for them.

For the detailed path from symptoms to mechanisms, continue with [Where slow members enter the foreground path](@/posts/bcachefs-slow-member-mechanisms.md). For the stranger swap work and the ways the experiments themselves failed, see [Why experimental bcachefs swap is such a useful stress test](@/posts/bcachefs-swap-and-testing.md).
