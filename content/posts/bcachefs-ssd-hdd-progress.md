+++
title = "Towards full SSD performance in mixed SSD/HDD bcachefs"
date = 2026-09-05T08:58:00+08:00
draft = false
description = "Progress on keeping SSD foreground work responsive while bcachefs uses HDDs in the background: durability, scheduling, readahead, swap, failed experiments, and what remains unsolved."
+++

[Bcachefs](https://bcachefs.org/) is a copy-on-write filesystem for Linux. One of its attractions is that a single filesystem can span a mixture of devices: fast NVMe SSDs, older SATA SSDs and large spinning hard disks, rather than requiring a matched set of drives.

Its [foreground and background targets](https://bcachefs.org/Caching/) let you use that mixture as a storage hierarchy. New writes can go to the SSDs, with data moved to the HDDs in the background. The SSD copies can stay around as a cache. In principle, that gives you SSD responsiveness with HDD capacity.

Alas, placing foreground data on SSDs does not yet reliably isolate it from the slow devices. Foreground writes can still be held up by filesystem-wide work that waits for the slowest member, dragging their latency down to HDD speeds—or worse when a drive stalls. The data need not be written to that HDD for it to delay the operation.

Slow writes are not the only symptom I have encountered. In [the earlier investigation](@/posts/frozen-ls.md), ordinary `ls`, `stat` and `grep` commands could stop for thirty to sixty seconds or more during heavy background writes. Later, I also saw small-file reads stall and `syncfs` take far too long to return. These are not necessarily one bug: lock contention, read-ahead, writeback and durability dependencies can each turn background activity into a foreground pause. This work is about finding and removing those dependencies, not just improving a bulk-write benchmark.

This is a progress report. The machine is running an experimental bcachefs module and has moved hundreds of GiB to the HDDs. There are also still unexplained foreground stalls. Some fixes have good independent reproducers and clean review branches; others are prototypes; several attractive explanations have been withdrawn.

This post covers the larger investigation that followed, including the mistakes. I have used coding agents extensively to inspect code, build candidates, run experiments and review the results. Their ability to keep working is useful. Their ability to write a confident explanation is not evidence that the explanation is right.

For orientation, the physical machine currently runs a stock Arch kernel with only bcachefs replaced. Reconciliation—the background work that brings data into line with placement, replication and compression policy—is enabled. CopyGC is disabled. Both the per-device readahead setting and aggregate cap are 2 MiB; the mover has two I/Os and 1 MiB of outstanding-work allowance. The most recent recorded snapshot showed about 475 GiB moved since mount. The newly tested CopyGC and write-buffer series, adaptive readahead, generic block-layer changes and bcachefs swap are not deployed there.

## What I am trying to achieve

The priorities matter, because it is easy to spend a week optimising the wrong thing.

First, protect SSD foreground latency. I do not need a zero-percent regression. I can live with some lost throughput or a modest latency increase. I do not want an SSD operation to become a seconds-long wait because an unrelated HDD is busy.

Second, make background work genuinely useful without catastrophic failure modes. A test that keeps the foreground fast by preventing all movement is not a success.

Third, eventually use as much idle HDD capacity as possible. This is a worthwhile stretch goal, not a prerequisite for an initial improvement.

Fourth, later, I want a desktop mode that can complete work at RAM speed, including a carefully bounded relaxation of `fsync`. More on that below. The current work must preserve normal durable acknowledgement; “make it faster” is not permission to lose committed data.

These are related goals, but they are not the same benchmark.

## Why “just give background I/O low priority” is insufficient

There are several places where an operation can wait:

1. Before it obtains a request slot.
2. In the scheduler, after it has obtained one.
3. After the driver has submitted it to the device.
4. In the filesystem, waiting for some other operation that holds a needed resource.

The elevator mostly operates in the second of those places. Request slots are usually called *tags* in the block-layer code. It cannot prioritise a foreground request that has not yet acquired a slot. It cannot pull a command back out of a drive's firmware. And it does not know that a journal entry needed by an SSD writer is waiting on an unrelated HDD operation.

The journal is especially important. It is a finite, ordered resource. If an early entry cannot finish, later work eventually runs out of room behind it. A local slow-device dependency can become a filesystem-wide pause.

That has been the most useful organising model: find the paths by which foreground progress becomes dependent on a slow member. It is not a universal diagnosis. A task stack showing a journal wait does not identify which device, operation or resource caused it.

### Bound outstanding work, not just bandwidth

An early direction was to pace the mover. This remains useful, but a fixed bandwidth ceiling is a poor expression of the goal. I want idle capacity used; I want little enough work outstanding that newly arriving foreground work has a chance.

The physical setup currently limits the mover to two I/Os and 1 MiB in flight. Those are outstanding-work limits, not a promise to move at only some fixed number of bytes per second. Earlier testing used deliberately low artificial-disk rates; those test settings were not my desired HDD performance.

The experimental mover accounting itself needed attention. Completed mover reads could escape the credit that was supposed to cover them before their corresponding writes had been submitted. The accounting needed to cover that interval too.

There is also generic block-layer work: idle-request admission, scheduler-tag headroom, and requeue accounting. Requeued requests must not keep consuming an allowance as though the driver still owns them. These are separable from bcachefs. The current stock-kernel-plus-module dogfood does not magically include local changes to the generic block layer.

## Durability without waiting for every drive

One important coupling was the journal preflush. The examined code could flush every writable member. A user-data HDD could therefore delay an SSD-only commit even when that HDD owed no durability relevant to it.

The scoped-preflush work in the dogfood stack tracks per-device durability obligations and scopes the flush accordingly. The qualification must work in both directions: a debt-free slow member should be excluded, but a member whose writes are required must still be waited for. An injected flush failure must not turn into a successful durable acknowledgement.

This is also why byte counters alone are inadequate evidence. An empty flush transfers no user-data bytes and can still take a long time.

Background demotion introduces another ordering problem. Suppose there is a good SSD copy and I want to move responsibility to an HDD copy. The SSD must not stop being authoritative before the HDD copy is durable. An early local design made every small extent individually durable with FUA—Force Unit Access—which proved expensive. Batching the durability work can be much more productive, provided the old copy stays authoritative until the batch is safely committed.

The batching machinery then needs a real lifecycle: cancellation, device removal, read-only transitions, restart, and device identity changes. One local bug was particularly unglamorous: a flag in bit 16 travelled through a 16-bit field and disappeared. That was a bug in the experimental stack, not an upstream finding to take credit for.

The lesson is not that flushing is bad. It is that durability obligations need explicit ownership. Removing a wait is correct only when the thing being waited for is genuinely unnecessary, or when another protocol preserves the obligation.

## Superblocks turned out to be part of the foreground path too

Not all slow-member I/O comes from moving file contents. Replica-layout maintenance and allocation/repair metadata can cause superblock writes. If those writes synchronously visit a slow member, they can reintroduce the dependency that the data path was designed to avoid.

The experimental stack includes reducing replica-layout churn and scoping some metadata updates to the appropriate metadata devices. A more radical idea—making HDD superblocks merely advisory—remains an open protocol question. Membership, stale copies and recovery authority are exactly where a convenient performance shortcut can become a durability bug.

This investigation also found a more specific crash-recovery coverage problem. A bitmap intended to help repair locate metadata could omit an old btree node that was still authoritative after a crash. It reproduced on a control excluding the new scoped-superblock patch; that control still had earlier local changes, so the supported claim is that the bug predated that patch at the tested base, not that it affected every upstream release. The newest runtime allocation view and the set of locations recovery might need are different things.

Keeping every historical bit forever avoids one kind of omission but eventually makes the allocation information poor. The experimental answer separates exact runtime allocation state from conservative persistent repair coverage.

I also had to retract an overly neat causal story here. Removing one superblock path did not prove that path caused a particular catastrophic stall. Some early “passing” experiments had not verified that delayed I/O was actually outstanding. Another path—an all-member preflush—could still cause a stall with no slow-member superblock bytes. The code really could wait for a slow member during a superblock update; that did not establish the cause of every observed failure.

## Readahead: the favourable case matters too

A five-member filesystem had a 10 MiB aggregate readahead window because the implementation summed a 2 MiB allowance per device. That can make sense when useful reads fan out across several devices. It is less attractive when the file being faulted in has one relevant slow replica.

Optional read-ahead can then fill request slots and make a demand operation wait to obtain one. In the targeted single-slow-replica VM shape, reducing the window from 10 MiB to 128 KiB removed the observed tag waits and reduced maximum completions from roughly 2.4 seconds to 40 ms in two retained runs. Those maxima describe the tested jobs, not a bound on future operations. A moderate cap alone was not sufficient in every shape.

The obvious next step was not simply to revert the summing behaviour forever. A striped positive control showed that stock's larger window really could help. A policy that wins only by disabling useful parallel reads has hidden its trade-off.

There are two distinct contributions here. One is straightforward: changing upstream's runtime `dev_readahead` option did not recalculate the mounted filesystem's window. A runtime-settable knob should actually take effect. The other is policy: an aggregate cap, and an experimental access-/selected-device-aware path that avoids waiting for scarce slots for optional reads while retaining useful fan-out.

The cap is in the physical setup. The more adaptive policy is still experimental. Neither amounts to a general guarantee that an HDD-only demand read will be fast.

## Placement must survive maintenance

Another independent problem is CopyGC, the compactor that evacuates live data so buckets can be reused. Setting foreground and background targets is not enough if a maintenance path chooses destinations outside that policy.

A dedicated stock-upstream VM reproducer showed CopyGC putting an SSD-resident file on the HDD with reconciliation disabled. The filesystem could remain structurally valid while violating the performance placement I wanted.

On upstream snapshot `42cb08fe01fa`, tested on 5 September, the control completed 1,378 evacuations and acquired a durable HDD pointer. The two-patch treatment completed 521 evacuations while retaining source-SSD placement. A separate two-SSD case checked that real compaction occurred while each SSD retained its durable replica: it completed 1,208 evacuations and changed many physical pointers, with none moved to the HDD.

All three cases passed readback, clean unmount and offline fsck. The different evacuation counts are not a throughput comparison; I have not characterised a performance cost from them.

That is useful evidence for a small independent series, not an exhaustive claim about every device-removal, exhaustion or erasure-coded layout. The second part of the design is handling destination exhaustion sensibly; restricting destinations creates legitimate cases where compaction must back off.

CopyGC remains disabled on my physical filesystem. Its VM-tested fix has not yet been deployed there.

Target semantics produced a separate surprise. In the upstream behaviour investigated, `background_target=none` inherited the foreground target rather than meaning unrestricted placement. That made an old “temporarily disable the background target” boot precaution particularly misleading: it did not mean what the prose around it said.

My preferred interface would distinguish `none`, `inherit` and an explicit target. There are compatibility details, including the possibility that someone already has a device group named `inherit`. A practical workaround is to put the device labels under a common parent such as `all.ssd` and `all.hdd`: `all` then explicitly names the whole set. It is not a way to disable reconciliation, which has its own control.

## Swap has been a useful source of difficult workloads

Swap is not just another file-writing benchmark. The system may need to finish swap I/O in order to free the memory that other work needs. If that completion itself needs an unavailable filesystem or memory resource, ordinary I/O can become a reclaim deadlock.

Resurrecting the bcachefs swap work has exposed a collection of separate problems across upstream code and local experimental branches; this is not a list of bugs all present in stock: lost reclaim context, request-slot exhaustion, missed transaction-cache wakeups, write-buffer accounting, optional allocations that block, and completion paths that need resources after admission.

Some early defensive mechanisms were not supported by ablation. Removing one safeguard could reproduce a stall; removing another did not show a benefit in that test. “The armoured branch survived” is not evidence that every piece of armour was necessary. Conversely, fixing the first demonstrated problem did not make the whole swap path safe.

A particularly productive recent result is the write-buffer reservation series. It fixes an upstream write-buffer units error, reserves space before commit, validates an entire batch before conversion, and keeps reservation ownership attached to journal buffers. Recovery made that last point concrete: deciding ownership from the filesystem's current replay phase is wrong if an operation crosses the transition from replay to normal running.

A directed test made the old transition fail and the corrected version recover. The clean series also has unit tests, metadata stress, swap/memory-pressure composition, unclean-cut recovery and ordinary dirty-writeback tests. In three paired writeback blocks, the treatment completed about 10% fewer, 4% fewer and 2% more writes than its paired parent control. Here fewer writes means less workload progress during the pressure test. That rules out the gross collapse the screen was designed to detect; it is not evidence of zero performance cost.

This series is ready for human review. It is not yet in the physical dogfood and does not solve all swap liveness.

The generic swap work has its own dead ends. Reserving address clusters looked like a route to guaranteed allocation headroom, but long-lived unrelated entries can keep those clusters occupied. A detached pool for the required table storage is a different prototype. Separately, headroom-aware swap routing can help choose where to send work, but choosing a backend does not guarantee that backend can finish under reclaim pressure.

Bcachefs swap remains VM-only in this effort.

## Explicit sync and shutdown need their own tests

A long `syncfs` is a different problem from a slow individual write. Continuous redirtying can make a wait for outstanding writeback keep encountering new work. The writeback-boundary prototype brackets the wait for an inode so later submissions cannot extend it indefinitely. This needs a VFS change as well as a bcachefs companion; it cannot be delivered solely by replacing a filesystem module.

A VM test combined mmap redirtying with direct writeback requests on the same inode. The prototype returned from `syncfs` in about 1.18 seconds on a 50 ms delayed device while both producers remained alive. A hard reset immediately afterwards recovered the acknowledged 64 MiB prefix exactly and passed offline fsck. That is a specific safety and liveness result. A remaining same-file `fdatasync` cost reached about 2.5 seconds in another test, so bounding one wait can move pain elsewhere.

One suspected concurrent-caller race was a dead end: source inspection and an instrumented overlap test showed that the VFS already serialises the relevant intervals. Adding another mutex would have duplicated that guarantee. The writeback-boundary interface remains review work, not a deployed general cure for long syncs.

Shutdown also found separate edges. Delayed discard must not keep emergency shutdown waiting indefinitely. A normal request to stop reconciliation must leave the worker able to restart rather than leak an internal walk sentinel as an error. These lifecycle cases need explicit tests even when steady-state read and write tests pass.

## What the failed boots taught me

Several earlier dogfood boots went badly, sometimes without any deliberate benchmark. There were hung tasks, journal waits, invalid-key and reconciliation diagnostics. One candidate accidentally included an experimental swap lineage even though no bcachefs swapfile had been activated. The resulting diagnostics are evidence that the candidate was unacceptable, not a clean experiment identifying one responsible patch.

There were also mistakes that were not filesystem algorithms at all. A broad custom kernel made NVIDIA and input-device differences part of the experiment. Those problems had not appeared in the earlier pre-mortems. I now prefer the distribution kernel with an exact external bcachefs replacement, keeping the unrelated modules as close to stock as possible.

A known reconciliation stop/restart fix was omitted from a later candidate. Testing a fix once is not enough; the reproducer must run against the exact candidate that is about to boot. The missing fix was a reminder to check the candidate's contents as well as its identity.

The same applies to the boot contract: the embedded module, root mount options, persisted target names, early services and eventual runtime policy must agree. Testing a sysfs transition in a fresh VM is not the same as booting with the production-like persisted configuration. Most of this can be exercised in VMs, with UEFI-specific tests reserved for the firmware path.

One-shot boot selection and automatic fallback did work during a failed attempt. The safety mechanism worked; the candidate failed. A machine that stays booted but makes basic operations wait for minutes is not healthy merely because it has not panicked.

I now keep successive pre-mortems rather than rewriting the first one after seeing the outcome. Broad hazard coverage deserves some credit; naming the symptom deserves more; predicting the mechanism requires evidence. Later revisions can also be worse than earlier ones if a few green experiments make the language overconfident.

## The tests were part of the problem

I have spent too much time on apparatus failures. They deserve to be described, because otherwise a list of successful experiments gives a false impression of how direct the progress was.

Some tests looked for `bcachefs` in each log line. Relevant diagnostics include continuation lines without that prefix. The replacement classifies complete multiline records into related, unrelated and needs-review categories, with explicit failure signatures. Zero classified failures still does not mean every warning is irrelevant.

Other false leads included an apparent disk wedge that was only a changed delay table, tests that did not prove useful background movement, and a tmpfs quota exhaustion that made fake devices fail. `df` alone did not establish that an experiment could allocate its backing store.

VM disks initially backed by the working bcachefs filesystem were also a moving target: that host filesystem and its patches were under active investigation. The observations from those tests still matter, but I want smaller reproducers on stable backing. I use tmpfs for disposable media and a separate ext4 filesystem for durable evidence.

The right response to a repeated harness mistake is to change the harness, not promise to be more careful next time. Exact module checks, parameter readback, useful-progress checks, bounded lifecycle observation and tests of the checks themselves have become reusable parts of the workflow.

I also do not want “wait for a pristine machine” to become a substitute for experimental design. Short randomised blocks, paired controls and repeated fresh guests can be more productive. Concurrency is something to measure and account for, not either forbid or assume free.

Natural workloads—kernel builds, working sets that do and do not fit in the SSD tier, and swap pressure—are useful screens. They do not replace directed reproducers. A combined regression workload may tell me only that one of several things broke; that is enough to reject a candidate. The standalone cases are what let me diagnose it afterwards.

## A current example: scan metadata competing for SSD capacity

The latest investigation concerns reconciliation scanning itself. It can rewrite metadata before it moves any user data.

On the physical machine, one active-then-paused observation showed p99 changing from about three seconds to twenty milliseconds. It was tempting to declare the scan responsible. But the periods differed, device state was changing, and the comparison was not controlled. Earlier large-tree VMs produced thousands of metadata writes without the seconds-long effect. Removing prefetch or adding a scan gate did not consistently help there.

A better question was whether the fake device model represented the contested resource. In this experiment that resource is the SSD's own queue: scanner metadata writes compete with foreground writes there, even while HDD user-data movement is blocked.

`dm-delay` can delay a bio before submitting it to the underlying queue. That need not occupy an underlying driver slot during the delay. Compare it with a device that holds four slots until their delayed completions. Both can give an isolated request roughly 2 ms latency, yet behave very differently under load.

At queue depth 32, a block-only calibration measured roughly 16,000 IOPS with pre-submission delay versus 2,070 with four-slot delayed completion. Equal single-request latency was not equal service capacity.

I then put the populated-filesystem scan on two models using memory-backed null devices with four slots. The pre-submission model adds a `dm-delay` layer, and setup takes different lengths of time in the two models; those differences remain in the comparison. Three randomised blocks each contained both models and both scan states, with a fresh guest for every cell. Each cell writes 256 one-MiB requests: its p99 describes roughly the three slowest writes, not a well-estimated rare-event rate. With delayed completion, active-scan p99 was about 47–49 ms versus 14–17 ms paused. With pre-submission delay, active p99 was about 3–4 ms versus 8–9 ms paused.

The pre-delay arm is faster with scanning active, an inversion I have not explained. That limits how I interpret the otherwise repeatable contrast. It is still not the physical seconds-long stall. The largest completion across these blocks was about 492 ms, in a paused cell; every block's maximum was paused. The experiment also shares one established image, and the foreground writes a replica to the delayed SSD. This is not a pure NVMe-only foreground test.

Recorded scanner stacks show tag waits on a deferred metadata write submitted when a transaction unlocks along the prefetch/node-fill path. They are not foreground-thread backtraces, and the monitor includes setup and post-job capture, without explicit shared-clock markers delimiting the timed job. The next step is to discriminate the mechanism and confirm important findings on fresh images and current upstream, not declare the old prefetch patch vindicated.

I would eventually like a small simulation of these resource and device models, calibrated against real measurements and checked on held-out workloads. A simulator that reproduces only the assumptions I put into it would be a more elaborate verbal explanation, not independent evidence.

## Real hardware is less polite

At least one of my HDDs uses drive-managed shingled magnetic recording. Its behaviour can depend on previous writes and internal cleaning. A constant 20 ms delay cannot capture that. Elevator ordering can still help arrange requests before firmware takes ownership; it cannot eliminate those internal cleaning intervals.

There have also been SATA command timeouts and link recovery. In one retained incident a small read failed after roughly 34 seconds; bcachefs retried another replica and repaired the affected copy. That is valuable resilience evidence. It does not identify whether the original fault was the drive, cable, power or controller.

A useful filesystem should contain the damage from a bad slow member where the data layout permits it. It cannot promise SSD latency for data whose only usable copy is on an unresponsive disk. Nor can it create unlimited SSD space: if foreground allocation really needs demotion to free room, that is a dependency requiring headroom and watermarks, not another scheduler preference.

Scrub exposed more work for later: an “uncorrected” counter that needs careful interpretation, and whether more device-ordered reading could reduce seeks without excessive buffering or interference. Fuzzing the relevant accounting and recovery paths is on the list. A partial scrub is not proof that every byte is readable.

## Desktop mode comes later

By desktop mode I mean making interactive work fast, including returning from `fsync` before persistence where I have deliberately opted into that contract.

A few seconds of lost work can be practically invisible to me as a human. A minute is a useful upper bound to explore, not a fixed chosen period. The hard requirement is consistency: after a crash, recover a state consistent with one recent global point, not an arbitrary mixture of each application's last surviving writes.

That is stronger than independently buffering each application's saves. Ordering hints from explicit sync operations may still matter even when their durability acknowledgement is relaxed. Memory pressure, crashes during checkpoint publication and the boundary of the promised loss interval all need a protocol and a test oracle.

I would also accept a temporary single durable replica for new data, with background work restoring the usual two. That might help drain RAM under pressure. But “one durable copy”, “two durable copies” and “only in RAM” must remain distinct states with explicit repair obligations.

None of this should change server mode. If the filesystem tells another computer that data is permanent, the data must be permanent. Human tolerance for losing a recent keystroke is not a licence to weaken an ordinary durability acknowledgement.

## Where this leaves the work

There is real progress: demonstrated slow-member coupling, smaller independent reproducers, useful physical background movement, and separable fixes for durability, placement, lifecycle and reservation ownership.

There is also a substantial remaining gap between that and “adding HDDs does not noticeably hurt my SSD work”. Rare physical stalls, recurring bad-device behaviour, capacity pressure and the complete composition of the latest fixes still need work. Testing one review-ready series does not qualify the whole desktop stack.

The most important process change is keeping the evidence separate from its interpretation. Raw results stay. Explanations get dated corrections, and old claims remain visible when they are superseded. An occasionally reread log is less convenient than a tidy success story, but it makes it harder for a disproved explanation to become received wisdom.

The aim is still the same ordinary one: let the slow disks be useful without making everything else wait for them.
