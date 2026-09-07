+++
title = "Why experimental bcachefs swap is such a useful stress test"
date = 2026-09-07T15:20:00+08:00
draft = false
description = "Context for experimental swap-on-bcachefs work, the liveness bugs it exposes, and the testing failures that changed the mixed-tier investigation."
+++

This is a companion to [Towards full SSD performance in mixed SSD/HDD bcachefs](@/posts/bcachefs-ssd-hdd-progress.md). The filesystem and block-layer details are in [Where slow members enter the bcachefs foreground path](@/posts/bcachefs-slow-member-mechanisms.md).

## What “bcachefs swap” means here

Linux commonly swaps to a dedicated block partition. It also supports swapfiles on filesystems which can provide a stable mapping from swap offsets to disk blocks. Once activated, those ordinary swapfiles mostly bypass the filesystem's normal buffered I/O path.

The bcachefs work in this project is more exotic. Bcachefs is copy-on-write and moves extents around; the experimental swap implementation lets the filesystem itself translate and service swap I/O. It is development work, not a feature deployed on my physical filesystem, and not something an ordinary bcachefs user should infer from seeing a swapfile on another filesystem.

I resurrected the branch partly because I want swapfiles on bcachefs eventually, but also because it is a remarkably effective stress workload for the main SSD/HDD goal.

Swap participates in memory reclaim. The kernel may need to write a swap page before it can free the memory that another operation needs. If the swap completion path itself tries to allocate memory, enter the same filesystem, obtain an exhausted request slot or wait for a resource held by reclaim, an ordinary performance problem can become a liveness cycle.

That is stronger pressure than “write a large file while compiling”. It forces the storage stack to make progress precisely when memory and I/O resources are scarce.

## What it exposed

The swap lineage exposed separate issues across upstream code and local experiments: lost reclaim context, request-slot exhaustion, missed transaction-cache wakeups, write-buffer accounting, optional allocations which block, and completion paths which need resources after admission. This is not a claim that every item exists in stock bcachefs.

Some early defensive mechanisms did not survive ablation. Removing one guard reproduced a stall; removing another showed no benefit in its tested shape. “The armoured branch survived” is not evidence that every piece of armour is needed. Conversely, fixing the first demonstrated cycle did not make the whole swap path safe.

One productive result is the write-buffer reservation series. Bcachefs stages some btree updates in a write buffer before converting them into their final form. The series fixes an upstream units error, validates an entire batch before conversion, reserves capacity before commit and attaches reservation ownership to the journal buffer which actually owns the work.

Recovery made the ownership issue concrete. Deciding whether a reservation is needed from the filesystem's current replay phase is wrong if a transaction begins during replay and finishes after the transition to normal running. A directed test made the old transition fail and the corrected version recover.

The clean series also has focused unit tests, metadata stress, swap and memory-pressure composition, unclean-cut recovery and ordinary dirty-writeback tests. In three paired writeback blocks, the treatment completed about 10% fewer, 4% fewer and 2% more writes than its parent control. That rejects the gross collapse the screen was designed to catch; it is not evidence of zero performance cost.

Generic swap work produced its own dead ends. Reserving address-space clusters looked like a route to guaranteed allocation headroom, but unrelated long-lived entries can keep those clusters occupied. A detached pool for required table storage is a different prototype. Headroom-aware routing can choose a less pressured swap backend, but choosing one does not prove that it can finish under reclaim.

Bcachefs swap remains VM-only in this effort. Headroom-aware ordinary swap selection is a separate patch line and can be tested without activating bcachefs swap.

## The failed dogfood boots mattered

Several physical dogfood boots went badly without any deliberate benchmark. There were hung tasks, journal waits, invalid-key and reconciliation diagnostics. One candidate accidentally included an experimental swap lineage even though no bcachefs swapfile had been activated. The resulting diagnostics proved the candidate unacceptable; they did not cleanly identify which patch was responsible.

Other failures were not filesystem algorithms. A broad custom kernel made NVIDIA and input-device differences part of the experiment. The keyboard and pointing-device trouble had not appeared in the pre-mortem. I now prefer a stock distribution kernel with an exact external bcachefs replacement, keeping unrelated modules and firmware behaviour close to stock.

A known reconciliation stop-and-restart fix was omitted from another candidate. Testing a fix once is insufficient. Every quick known reproducer should run against the exact composition about to boot, and the package must prove which patches, module and kernel it contains.

The boot contract is broader than a module hash. Embedded module, root mount options, persisted target names, early services and eventual runtime policy must agree. One candidate failed because a placement option which had become sysfs-only was still supplied as a mount option. Most of that contract can be tested in an ordinary VM; a separate OVMF test covers the UEFI path.

One-shot boot selection and automatic fallback did work when a candidate failed. That was a useful success of the safety mechanism, not a success of the candidate. A machine that remains booted while basic operations wait for minutes is not healthy merely because it has not panicked.

## The experiments also failed

Some early log checks looked for `bcachefs` on each line. Relevant diagnostics include continuation lines without that prefix. The replacement parser groups complete multiline kernel records and classifies them into related, unrelated and needs-review categories. Zero definite failures still does not imply that every warning is harmless.

Other false leads included an apparent disk wedge caused by a changed delay table, tests which did not prove useful background movement, and tmpfs quota exhaustion which made fake devices fail. `df` alone does not prove that an experiment can allocate its backing store.

VM disks were initially stored on the host bcachefs filesystem under active development. The recursion itself can be an interesting integration test, but it makes the host an unstable experimental substrate: a small host fix can silently change the guest's apparent device behaviour. Disposable media now defaults to tmpfs, with compact durable evidence copied to a separate ext4 rescue filesystem.

The kernel-log classifier, exact module verifier, reusable device models and lifecycle gates came from these failures. The principle is simple: best is to automate the right operation; second best is to automatically detect that the wrong operation happened.

That does not mean waiting for a pristine host. Realistic interference is valuable. Short randomised blocks, paired controls and repeated fresh guests are often more informative than postponing work until the desktop is idle. Competing processes, device counters and all individual observations are retained. A noisy block may fail to estimate one treatment effect while still revealing a real integration failure.

Natural workloads—kernel builds, working sets which do and do not fit on SSD, and swap pressure—are useful screens. A combined A+B+C test may only establish that something failed before reaching C; that is enough to reject a candidate. Standalone reproducers remain necessary for diagnosis and upstream review.

## Evidence and interpretation are separate logs

The investigation keeps an append-only evidence ledger and a separate reasoning ledger. Raw outputs, exact commands, hashes and individual observations remain durable. Interpretations receive dated corrections rather than silently replacing the previous explanation.

That matters because several confident stories were wrong. A superblock ablation showed that a slow-member write path existed, but did not isolate it as the cause of a particular journal wedge. A scan-active physical period was dramatically slower than a paused period, but the periods were not comparable enough to establish causality. A fake device with the right queue-depth-one latency had completely wrong saturated service capacity.

Successive pre-mortems are also preserved. Broadly imagining “activation trouble” is better than ignoring boot risk, but it should not receive the same credit as predicting an obsolete mount option. Later pre-mortems can become worse if a few green tests create unjustified confidence, so rewriting the old one after learning the outcome would destroy useful evidence.

I would eventually like a small simulation of the shared resources and device models, calibrated against measurements and tested on held-out workloads. A simulator which merely reproduces the assumptions put into it is an elaborate verbal explanation. Its value would come from making quantitative, falsifiable predictions which survive cases not used for tuning.

## Desktop mode is a later contract

The swap work connects to a longer-term desktop mode, but the two should not be confused. Desktop mode means deliberately allowing interactive work to return at RAM speed, including a bounded lie about `fsync` for a user who opts in.

A few seconds of lost work can be practically invisible to a human; roughly one minute is an upper design bound to explore. The hard requirement is consistency after a crash: recover a state corresponding to one recent global point, not an arbitrary mixture of each application's last surviving writes.

I would also accept temporarily writing one durable replica rather than the usual two, with background work recording and repairing the debt. “One durable copy”, “two durable copies” and “only in RAM” must remain distinct states.

None of this changes server semantics. If a filesystem tells another computer that data is permanently committed, it must be permanently committed. Human reaction time is not a licence to weaken an ordinary durability acknowledgement.

For the current outcome and priorities, return to [the short progress overview](@/posts/bcachefs-ssd-hdd-progress.md). For the concrete journal, readahead, CopyGC and reconciliation mechanisms, read [Where slow members enter the bcachefs foreground path](@/posts/bcachefs-slow-member-mechanisms.md).
