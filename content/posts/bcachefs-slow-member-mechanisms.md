+++
title = "Where slow members enter the bcachefs foreground path"
date = 2026-09-07T15:15:00+08:00
draft = false
description = "The journal, durability, superblock, readahead, reconciliation and CopyGC mechanisms behind SSD stalls in a mixed bcachefs filesystem."
+++

This is the technical companion to [Towards full SSD performance in mixed SSD/HDD bcachefs](@/posts/bcachefs-ssd-hdd-progress.md). A third piece covers [experimental bcachefs swap and the testing process](@/posts/bcachefs-swap-and-testing.md).

The central problem is easy to state: data intended for the foreground SSD tier can still wait for a slow HDD elsewhere in the filesystem. The interesting question is not merely which device received bytes, but which completion or shared resource the foreground path depends on.

## Scheduling happens at several layers

There are several places where an operation can wait:

1. Before it obtains a request slot.
2. In the block scheduler, after it has obtained one.
3. After the driver has submitted it to the device.
4. In the filesystem, waiting for another operation that holds a needed resource.

The elevator mostly operates in the second of those places. Request slots are usually called *tags* in the block layer. The scheduler cannot prioritise a foreground request that has not yet acquired a tag. It cannot pull a command back out of a drive's firmware. It does not know that a journal entry needed by an SSD writer is waiting on an unrelated HDD operation.

That is why changing the NVMe or HDD scheduler can help without being a complete solution. Low-priority background I/O needs bounded ownership of scarce resources before, during and after scheduler admission.

## Bound outstanding work, not just bandwidth

An early direction was to pace the background mover. This remains useful, but a fixed bandwidth ceiling is a poor expression of the goal. Idle HDD capacity should be used; the important constraint is leaving little enough work outstanding that newly arriving foreground work has a chance.

The physical setup currently limits the mover to two I/Os and 1 MiB in flight. These are outstanding-work limits, not a promise to move at only some fixed number of bytes per second. Earlier tests deliberately throttled fake disks to low rates; those numbers were properties of the experiments, not the desired production ceiling.

The accounting itself needed attention. Completed mover reads could escape the credit intended to cover them before their corresponding writes had been submitted. The allowance needs to cover the entire interval in which background work owns a resource or creates downstream work.

There is related generic block-layer work around idle-request admission, scheduler-tag headroom and requeue accounting. A request returned by the driver must not continue consuming an allowance as though the driver still owns it. These fixes are separable from bcachefs and cannot be delivered by replacing only the filesystem module.

## The journal amplifies local delays

The bcachefs journal is the recovery log which makes groups of filesystem updates crash-safe. It is finite and ordered. If an early entry cannot finish, later work eventually runs out of room behind it. A local dependency can then turn into a global pause even when the foreground data itself belongs entirely on SSD.

One important edge was journal preflush: before writing a commit, the filesystem asks devices to make relevant earlier cached writes durable. The examined code could flush every writable member. A user-data HDD could therefore delay an SSD-only commit even when that HDD owed no durability relevant to the operation.

The scoped-preflush work tracks per-device durability obligations and flushes the relevant set. Qualification must work in both directions: a debt-free slow member should be excluded, but a member whose writes are required must still be waited for. An injected flush failure must not become a successful durable acknowledgement. Byte counters alone are inadequate here because an empty flush transfers no user data and can still take a long time.

Background demotion introduces another ordering problem. Suppose an SSD holds the good copy and an HDD is meant to become authoritative. The SSD must not be released before the HDD copy is durable. An early design gave every small extent its own Force Unit Access write, asking the drive to make that command durable before completion. It was correct in spirit and painfully slow in practice.

Batching the durability work is much more productive, provided the SSD copy remains authoritative until the whole batch is safely committed. That creates a real lifecycle: cancellation, device removal, read-only transitions, restart and device identity changes all have to preserve the debt. One experimental bug was particularly mundane: a flag in bit 16 travelled through a 16-bit field and vanished.

The lesson is not that flushing is bad. Durability obligations need explicit ownership. A wait may be removed only when it is genuinely unnecessary or another protocol preserves the obligation.

## Superblock writes are not outside the foreground path

Not all slow-member I/O comes from moving file contents. Replica-layout maintenance and allocation or repair metadata can cause superblock writes. A superblock records filesystem-wide identity, membership and configuration rather than ordinary file contents. If its updates synchronously visit a slow member, they can reintroduce the dependency the data path was designed to avoid.

The experimental stack reduces replica-layout churn and scopes several metadata updates to explicit metadata devices. A more radical idea—making HDD superblocks advisory—remains a protocol question rather than a ready patch. Membership, stale replicas and recovery authority are exactly where a convenient performance shortcut can become a durability bug.

This investigation also found a crash-recovery coverage problem. Btrees are the on-disk indexes bcachefs uses for extents and other metadata. A bitmap intended to help repair locate that metadata could omit an old btree node which was still authoritative after a crash. The experimental answer separates exact runtime allocation state from conservative persistent repair coverage. Keeping every historical bit forever avoids one kind of omission but eventually makes the allocation information useless, so this is not a free monotonic fix.

I also had to retract an overly neat causal story. Removing one superblock path did not prove that it caused a particular catastrophic stall. Some early “passing” experiments had not verified that delayed I/O was actually outstanding. Another all-member preflush path could stall with no slow-member superblock bytes. The code really could wait for a slow member during a superblock update; that did not make it the cause of every observed failure.

## Readahead must preserve its good case

Readahead speculatively reads data beyond an application's immediate request in the hope that it will be needed next. Upstream bcachefs calculated its filesystem-wide readahead window by summing a per-device allowance. A five-member filesystem with 2 MiB per member therefore exposed a 10 MiB window to the VFS. That can be useful when sequential data is striped across several devices. It is unsafe as a prediction that any particular file will fan out evenly.

Optional readahead can fill request slots and make a demand read wait to obtain one. In a single-slow-replica VM shape, reducing the aggregate window from 10 MiB to 128 KiB removed observed tag waits and reduced maximum completions from roughly 2.4 seconds to about 40 ms in two retained runs. Those maxima describe those jobs, not a future upper bound.

A striped positive control then showed why blindly reverting the summed window was wrong: stock's larger window provided a substantial throughput benefit. A policy that wins only by disabling useful parallel reads has hidden its trade-off.

There are two contributions here. The straightforward bug is that changing the runtime `dev_readahead` option did not recalculate the mounted filesystem's VFS window. The policy work adds an aggregate cap and experiments with selected-device and access-aware admission. Optional reads should be declined rather than block for scarce tags, while reads that can use real device parallelism should still fan out.

The physical filesystem uses a 2 MiB cap. The adaptive policy has promising favourable and hostile controls but remains experimental. Neither guarantees that a demand read whose only good copy lives on an HDD will be fast.

## Reconciliation and CopyGC are different maintenance engines

Reconciliation scans extents and brings them into line with placement, replication, checksum and compression policy. It is the engine responsible for moving newly written SSD data towards the configured HDD background target and restoring missing replicas.

CopyGC is bcachefs's copying garbage collector. The filesystem allocates space in buckets. When a partially used bucket should be reclaimed, CopyGC moves its remaining live extents so the old bucket can be reused. It is compaction, not policy reconciliation, but it still chooses write destinations.

That distinction matters. Turning reconciliation off did not stop a stock-upstream CopyGC reproducer from moving an SSD-resident file to an HDD. The filesystem remained consistent while violating the intended performance placement.

The two-patch treatment tells CopyGC to preserve the source device and makes asynchronous destination exhaustion a reason to back off rather than spin or silently choose somewhere else. On the latest tested upstream snapshot, the untreated control evacuated 1,378 buckets and acquired a durable HDD pointer. The treatment evacuated 521 while retaining SSD placement. A separate two-SSD case evacuated 1,208 buckets and changed many physical pointers while retaining one durable replica on each SSD and none on HDD.

The exact current candidate later repeated the two-SSD shape: all 3,400 extents retained one pointer per SSD, CopyGC evacuated 1,218 buckets, and more than a thousand pointers changed on each SSD. Readback, clean shutdown and offline fsck passed. That is strong evidence for the tested placement invariant, not exhaustive coverage of surprise removal, erasure coding or arbitrary exhaustion.

## Reconciliation scanning may contend before moving data

The physical machine once showed foreground write p99 change from about three seconds with reconciliation active to twenty milliseconds with it paused, even though the scan moved no user data. It was tempting to declare scanning responsible. The periods differed, device state was changing, and the comparison was uncontrolled.

Large-tree VMs then produced thousands of metadata writes without reproducing the seconds-long effect. Removing prefetch or gating the scan did not consistently help. The better question became whether the fake device represented the contested resource.

`dm-delay` can hold a bio before submitting it to the underlying queue. That does not necessarily occupy an underlying driver tag during the delay. A device with four tags held until delayed completion has similar queue-depth-one latency but radically less saturated capacity. In a block-only calibration at queue depth 32, pre-submission delay delivered about 16,000 IOPS while completion-held delay delivered about 2,070.

Putting the populated scan on those models made active scanning slower in the completion-held model, but still did not recreate the physical catastrophe. Active-scan p99 was about 47–49 ms versus 14–17 ms paused; the largest completion in the blocks was about 492 ms and occurred while paused. The pre-submission model also showed an unexplained inversion where active scanning was faster.

Recorded scanner stacks include tag waits on deferred metadata writes submitted when a transaction unlocks along the prefetch and node-fill path. They are not foreground-thread backtraces, and the original monitor did not precisely delimit the timed interval. The evidence makes resource contention plausible; it does not yet identify the seconds-scale physical mechanism.

## Sync and shutdown create separate boundaries

A long `syncfs` is different from a slow individual write. Continuous redirtying can make a wait for outstanding writeback keep encountering new work. A writeback-boundary prototype brackets an inode's wait so later submissions cannot extend it indefinitely. This needs a VFS change as well as a bcachefs companion.

In one VM, `syncfs` returned in about 1.18 seconds on a 50 ms delayed device while mmap and direct writeback producers remained alive. A hard reset immediately afterwards recovered the acknowledged 64 MiB prefix exactly and passed offline fsck. Another same-file `fdatasync` still reached about 2.5 seconds, showing that bounding one wait can move the pain elsewhere.

One suspected concurrent-caller race was a dead end: the VFS already serialised the relevant intervals. An extra mutex would merely have duplicated that guarantee.

Shutdown found other independent edges. A delayed discard must not keep emergency read-only transition waiting indefinitely. Disabling reconciliation must leave its worker able to restart instead of leaking an internal walk sentinel as an error. These deserve explicit lifecycle tests even when ordinary read and write tests pass.

## Real drives add state

At least one physical HDD uses drive-managed shingled magnetic recording. Its behaviour depends on write history and internal cleaning. A constant 20 ms delay cannot capture that. Elevator ordering can still arrange requests before firmware takes ownership; it cannot eliminate a drive's internal cleaning pause.

The host has also seen SATA command timeouts and link recovery. In one retained incident a small read failed after roughly 34 seconds; bcachefs retried another replica and repaired the affected copy. That is useful resilience evidence but does not distinguish drive, cable, power, controller or firmware.

A filesystem can isolate SSD work from an unrelated bad HDD only where its layout provides another route. It cannot make the only usable copy of requested data respond quickly. Nor can it create unlimited SSD capacity: if foreground allocation genuinely requires demotion to free space, the dependency must be managed with headroom and watermarks.

The short [progress overview](@/posts/bcachefs-ssd-hdd-progress.md) describes what is running now. The final companion, [Why experimental bcachefs swap is such a useful stress test](@/posts/bcachefs-swap-and-testing.md), covers the unusually hostile workload and why several apparently clean results were apparatus failures instead.
