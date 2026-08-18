+++
title = "The fastest yes in the west"
date = 2026-08-17T22:40:00+08:00
description = "A code-golf challenge about printing y as fast as possible turned into a one-function Linux kernel patch: splice hands a pipe one buffer per 4 KiB page even from a 2 MiB folio. Removing that was worth 17x — and cost me four wrong explanations, the last of which was that the speedup was about pipes at all. Meanwhile the kernel community started removing the thing I was optimising."
+++

There is a [Code Golf Stack Exchange challenge from 2020][cg] called
*Fastest `yes` in the west*. You write a program that prints `y\n`
forever; you are scored on your source length times how much slower you
are than GNU `yes`. The accepted answer is a beautiful piece of
rule-lawyering: 112 bytes of C that writes 128 MiB and then repeatedly
calls the `FICLONERANGE` ioctl to double the *file's* length by
reflinking the same blocks, reaching a nominal 28 TB/s. It measures file
size, not bytes produced, and reflink means no bytes are produced at all.

[cg]: https://codegolf.stackexchange.com/questions/199528/fastest-yes-in-the-west

I wanted the honest version of the question. Not "how large a file can
you make appear" but: **how fast can a process actually hand `y\n` to
another process today?** No file, no reflink; the bytes have to arrive
somewhere real. On my desktop — an i9-13900K, kernel 6.17 — GNU `yes`
pushes about 80 GiB/s into `pv > /dev/null`.

The answer, after a day of measuring, is that the interesting bottleneck
is not in userspace at all, and that I was wrong about *why* three
separate times. This is the record of that, including the wrong parts,
because the wrong parts are where the method earned its keep.

## Userspace first: 209 GiB/s

The standard trick for a fast producer is `vmsplice`, which moves pages
into a pipe by reference instead of copying. Better still is to avoid
touching user memory at all: fill a `memfd` once with `y\n`, then
`splice()` from the memfd into the pipe forever. The kernel passes page
references; nothing is ever copied.

One extra ingredient matters enormously. `madvise(MADV_COLLAPSE)` on the
memfd turns its pages into 2 MiB shmem folios, which the splice path can
then hand over in bigger bites:

```c
int fd = memfd_create("y", 0);
ftruncate(fd, 8 << 20);
char *m = mmap(0, 8 << 20, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
for (long i = 0; i < (8 << 20); i += 2) m[i] = 'y', m[i + 1] = '\n';
madvise(m, 8 << 20, MADV_COLLAPSE);        /* 2 MiB folios */
for (;;) {
        off_t off = 0;
        while (off < (8 << 20))
                splice(fd, &off, 1, 0, (8 << 20) - off, 0);
}
```

That is **209 GiB/s** into an unmodified `pv`: 1.02 TiB in five seconds,
against 80 for GNU `yes`. Without the `MADV_COLLAPSE` it is 54 GiB/s, so
the folio size is doing most of the work.

I also re-ran [mazzo.li's `pipes-speed-test`][mz] — the reference for
"how fast are Linux pipes anyway" — on the same two cores, because
quoting his blog's 65 GiB/s against my hardware would have been
meaningless. Tuned up (vmsplice, huge pages, busy-loop reader) it reaches
145 GiB/s here at 64 MiB buffers. So the memfd approach leads the best
published userspace technique by about 1.4× like-for-like, which is a
much smaller and much more honest claim than the one I nearly made.

[mz]: https://mazzo.li/posts/fast-pipes.html

## The wall is one loop in the kernel

At 200 GiB/s both processes sit at roughly half a core, so nobody is
CPU-bound in userspace. `perf` says 69% of all cycles are in three
kernel functions: `splice_folio_into_pipe` on the producer side,
`page_cache_pipe_buf_release` and `__splice_from_pipe` on the consumer
side. All three are per-page.

Here is why, from `mm/filemap.c`:

```c
while (spliced < size && !pipe_is_full(pipe)) {
        struct pipe_buffer *buf = pipe_head_buf(pipe);
        size_t part = min_t(size_t, PAGE_SIZE - offset, size - spliced);
        *buf = (struct pipe_buffer) { .page = page, .offset = offset, .len = part, ... };
        folio_get(folio);
        pipe->head++;
        page++;
        ...
}
```

`part` is capped at `PAGE_SIZE`. A 2 MiB folio therefore becomes **512
separate `pipe_buffer`s**: 512 ring slots, 512 structure stores, 512
`folio_get`s, and later 512 releases on the other core. Large folios buy
the splice path almost nothing today; they amortise the page-cache
lookup and nothing else. The obvious change is to emit one `pipe_buffer`
spanning the whole folio — `offset` and `len` are relative to the head
page, and the generic consumers already walk subpages correctly on
64-bit, because `copy_page_to_iter` has a `page++` loop inside it.

That patch is about fifteen lines. Measured A/B in identical VMs, it took
the 2 MiB-folio case from 135 GiB/s to **2.3 TiB/s**, with the 4 KiB-folio
and `write(2)` paths unchanged as controls, and byte-exact output.

Then it fell apart in three different ways, and each one is more
interesting than the number.

## Wrong explanation #1: it isn't the refcount

I had a tidy story for the 17×: the `folio_get`/`folio_put` pair means
the producer and consumer cores ping-pong the folio's refcount cache
line, 512 times per folio. Collapsing to one buffer removes 511 of them.

Tidy, and false. I built the cheap version of that idea — keep per-page
buffers, but take all the references at once with a single
`folio_ref_add(folio, npages)` — and **pre-registered** what I expected
(1.5–2.5×) and what I would chase if it came out otherwise. It came out
at **1.1×**. Disassembly confirms the change is live: one `lock add`
where stock has 512 `lock incl`.

So the refcount traffic was never the bottleneck. The cost is the
per-page *machinery* — the ring slots, the loop, the per-buffer release
call — and only collapsing 512 buffers into one removes it. Writing the
prediction down first is what turned a disappointing measurement into a
corrected mechanism instead of a shrug.

## Wrong explanation #2: the number was measured on a broken pipe

A pipe has a size. `pipe(2)` gives you 64 KiB; `F_SETPIPE_SZ` can raise
it. Except the kernel does not track a pipe's size in bytes anywhere —
it tracks **ring slots**, on the assumption that a slot holds at most one
page. Put a 2 MiB buffer in one slot and every capacity check in the
subsystem is wrong.

I asked DeepSeek to refute the patch, which it did, three times, each
time predicting a number I could check. All three reproduced exactly:

| what | measured |
|---|---|
| splice repeatedly into a 64 KiB pipe | pinned **557 056** bytes (8.5× its size) |
| splice a 2 MiB buffer pipe-to-pipe into a 4 KiB pipe | destination held **2 MiB** (512×) |
| `F_SETPIPE_SZ` shrink a 2 MiB-holding pipe to 4 KiB | succeeded; pipe still held 2 MiB |

The third one is the instructive one: it is neither an "add" nor a
"move", it is a *budget change*, a category I had not thought to
enumerate. After the third round I stopped playing whack-a-mole and
grepped for every capacity and budget site in `fs/pipe.c`,
`fs/splice.c`, `mm/filemap.c`, `mm/shmem.c` and `kernel/watch_queue.c`.
The set is closed and small: seventeen checks that all route through
`pipe_full()`/`pipe_is_full()`, plus three places that assign
`max_usage`. Guard the *definition* and the callers inherit it.

So I added a maintained byte counter, `pipe->nr_bytes`, made "full" mean
byte-full as well as slot-full, made `F_SETPIPE_SZ` refuse a shrink that
would leave a pipe over its new budget, and taught pipe-to-pipe and
`tee()` to split an oversized buffer instead of moving it whole. A
`CONFIG_DEBUG_VM` validator recomputes the counter by walking the ring
and warns on mismatch, so a missed annotation shows up in testing rather
than drifting silently. (It has stayed silent, which is the only reason I
believe the counter.)

And with capacity actually enforced, the headline number changed. The
2.3 TiB/s had been measured on a pipe holding many times its nominal
size — and, it turned out, with the producer *busy-spinning* rather than
sleeping, because the wait predicate wasn't byte-aware either. The honest
numbers, capacity respected:

| pipe size | stock | patched |
|---|---|---|
| 1 MiB | ~140 GiB/s | ~210–307 GiB/s |
| 32 MiB | ~347 GiB/s | **8 226–9 712 GiB/s** |

Roughly **26× at a 32 MiB pipe**, and the win scales with the pipe's byte
budget, because that budget is exactly how many bytes may be in flight.
That relationship was invisible while the accounting was broken.

## Wrong explanation #3: my own fix deadlocked

With capacity enforced, a `read(2)` consumer took exactly one 1 MiB
buffer and then hung forever.

The cause is a pattern worth naming: **byte-aware blocking without
byte-aware waking**. A pipe holding one 1 MiB buffer in a 256-slot ring
is byte-full but not slot-full, and the reader only ever woke writers
when a *slot* freed:

```c
wake_writer |= pipe_full(head, tail, pipe->max_usage);
```

So the writer slept on byte capacity and nothing ever woke it. I audited
every place that *blocks*; I did not think to audit every place that
*wakes*. Both halves of a capacity rule need the same treatment.

My first fix for that was also wrong, in a way I want to keep on record
because it is so easy to repeat. I sampled byte-fullness once on entry
to the read function and reused it — but that function *sleeps in the
middle*, waiting for data, so the sample is stale exactly when it
matters. The kernel trace was unambiguous:

```
read exit: ret=1048576 wake_writer=0 was_bytes_full=0 head=1 tail=1 nr_bytes=0
wait_for_space sleep: head=513932 tail=513931 max=256 nr_bytes=1048576
```

The reader believed the pipe had never been byte-full; the producer was
sitting at exactly the budget. Sampling fresh, immediately before
releasing the bytes, fixes it. The full suite now passes: 64 MiB read
back byte-exact, 32 MiB through an `AF_UNIX` socket byte-exact, 16 MiB of
blocking pipe-to-pipe through a 4 KiB destination byte-exact, and every
capacity probe bounded.

## Wrong explanation #4: it isn't the ring either — it's the wakeups

Explanation #1 left me with "the cost is the per-page *machinery*". That
is close enough to sound finished, which is exactly the danger.

The obvious test is to hold the buffer format constant and vary only how
much the ring holds. On a stock kernel, with every buffer still one page,
I swept the pipe size for four different producers. `write(2)` went from
6.45 to 7.65 GiB/s across a 16× increase — flat, as a copy should be.
Every zero-copy producer scaled hard: `vmsplice` 29.6 → 160.6, `tee`
31.4 → 144.5, `splice` 31.9 → 213.3. And at the smallest ring all three
sat within 8% of each other, only separating as it grew.

So it is ring capacity, not per-page machinery. I wrote that down and
was wrong again within the hour.

`sendfile(2)` uses a pipe internally — `splice_direct_to_actor()`
borrows one from `current->splice_pipe`, hardcoded to 16 slots by
`PIPE_DEF_BUFFERS`, with no fd and therefore no way for userspace to
resize it. That looked like a gift: if capacity is the constraint, then
simply enlarging that one internal pipe would speed up `sendfile`,
`nfsd`, `ksmbd` and overlayfs copy-up, with none of the multi-page
buffers and none of the capacity accounting they force.

You can test that without touching the kernel. Do what `sendfile` does
internally — splice the file into a pipe, then splice the pipe onward —
but through a pipe you *can* resize:

```
sendfile, internal 16-slot pipe    281.25  282.98  283.02 GiB/s
manual, 1 MiB pipe                 282.29  277.24  259.15
manual, 64 KiB pipe                257.78  259.16  258.23
```

A 16× ring increase buys about **9%**. The same 16× bought 6.58× in the
sweep above.

The difference between the two experiments is that the sweep had a
separate consumer *process*. What ring capacity actually amortises is
producer/consumer **wakeups and context switches** — not per-fill work,
not slot machinery. Where the fill and the drain happen in one thread
inside one syscall, as they do inside `sendfile`, there is nothing to
amortise and the ring barely matters.

Which quietly demolishes the best justification I had. I had spent an
afternoon establishing that `splice_folio_into_pipe()` is reached from
`sendfile(2)`, `copy_file_range()`, nfsd, ksmbd and overlayfs copy-up,
and writing that up as the answer to "who would actually benefit".
Every one of those runs a single-threaded fill-and-drain loop. The
speedup is real, but it belongs to *cross-process* pipes: shell
pipelines, `yes | pv`, and FUSE daemons reading `/dev/fuse`.

The thing I had been measuring all along was the benchmark's own shape.

## Does any of this matter outside a benchmark?

Mostly not for throughput, and I would rather say so plainly than
oversell it.

Anything network-bound is unaffected: a 100 Gbit link is about 12 GiB/s,
and stock splice already does 140. NVMe tops out around 7 GB/s. If your
`sendfile()` web server or your NFS export is slow, this is not why. Nor
does it touch socket-to-socket proxying, where the buffers come from
network skbs rather than the page cache.

Where it does matter:

**CPU per byte, not bytes per second.** The patch removes up to 511 of
every 512 ring slots, buffer stores and release calls. For a server whose
CPU is shared between moving bytes and doing actual work, that is
returned cycles even when the pipe was never the bottleneck.

**Memory-speed IPC.** Pipelines that move data between processes through
tmpfs, `memfd` or the page cache — log shippers, local shuffles, data
loaders, anything using a pipe as a zero-copy conveyor — are exactly the
case where the pipe *is* the bottleneck, and where raising
`F_SETPIPE_SZ` now buys an order of magnitude instead of nothing.

**A tax that is growing.** Large folios in the page cache are becoming
the normal case rather than the exception. Every one of them currently
gets chopped into 4 KiB pieces on its way through a pipe, and that
overhead grows with folio size while the benefit stays flat.

## How much is zero-copy actually worth? About 2×, not 18×

Having been wrong four times about *why* the fast thing was fast, I
wanted a number for the more basic question: what does zero-copy buy
over just copying? So: the same transfer, same destination, once with
`sendfile` and once with `pread` + `write`, scored in CPU-seconds rather
than wall-clock, because for a real destination the destination sets the
pace.

| destination | sendfile | pread+write | ratio |
|---|---|---|---|
| `/dev/null` | 268.0 GiB/cpu-s | 14.6 | **18.4×** |
| AF_UNIX socket | 10.3 | 4.9 | **2.1×** |
| file → file | ~3.8 | ~3.4 | ~1.1× (noisy) |

Two things worth carrying away from that table.

The socket figure is the honest one, and it independently reproduces a
number from the kernel mailing list: Willy Tarreau, defending splice,
reported 62 Gbit/s per core with it against 31 without. That is 2.0×. I
got 2.1× by a completely different method on completely different
hardware. When two unrelated measurements of a contested quantity agree,
that is about the best evidence available.

And the gap between the first two rows is the real lesson. **A
microbenchmark with no destination cost overstates zero-copy by roughly
ninefold.** That single fact reconciles two camps who have been talking
past each other for years: the advocates quoting enormous speedups are
measuring to `/dev/null`, and Linus Torvalds saying zero-copy "has
seldom really been a huge advantage in practice outside of benchmarks"
is talking about the socket row. Both are looking at real numbers.

I had been quoting the `/dev/null` row throughout.

## The twist: they are removing it

While I was doing this, the kernel community started dismantling the
thing I was optimising.

A wave of vulnerabilities in 2026 — several found by LLMs — hit exactly
this machinery: page-cache pages handed into network buffers and then
processed in place. In response, Askar Safin posted a series that
removes `vmsplice()`'s zero-copy semantics entirely, turning it into a
plain copy. Christian Brauner applied it. Pedro Falcato added a sysctl
that forbids splicing to a file you cannot write, which [LWN][lwn]
describes as "an admission of defeat" — that splice cannot be
implemented in a way that prevents security vulnerabilities. Linus
Torvalds called `sendfile()` "a mistake" and said he wants to do the
same to it.

[lwn]: https://lwn.net/Articles/1075838/

GNU coreutils, meanwhile, removed the splice and `vmsplice` tricks from
`yes` and `cat`. LWN's Jonathan Corbet noted that "not many people see
`yes` as a performance-critical application". So the challenge that
started this and the kernel path that answered it were deprecated in the
same season, which I choose to find funny.

The obvious reading is that the patch is dead. I think the more useful
reading is that it changed genre. The open question in that thread is
whether splice's performance justifies its security cost, and it is
being argued with one real number — Tarreau's — against a lot of
intuition. A careful 2.1× that agrees with his, plus the observation
that the 18× everyone quotes is a measurement artefact, is worth more to
that discussion than the patch was ever worth to a merge window. A
measurement cannot be NAKed.

## What I would take from this

Three habits paid for themselves, all of them about being wrong faster.

**Ask a different model family to refute you, and make it predict a
number.** All three capacity bugs came from an adversarial review that
proposed a decisive measurement with each verdict. Predictions I could
run are worth vastly more than opinions I could only weigh — and I caught
one confident, detailed, *false* refutation the same way, by checking its
claim about `copy_page_to_iter` against the source and finding the
`page++` loop it said did not exist.

**Measure the naive version first, with your expectation written down.**
The cheap refcount-batching experiment cost one kernel build and
demolished my explanation of my own patch.

**An exit code is not a result.** My first attempt at running a
kernel-patch review bot exited 0 having reviewed nothing at all — every
call had failed internally. I only noticed because I read the log instead
of the status.

**The benchmark's shape is a variable, and it is the one you forget to
vary.** Four wrong explanations, and the last two were not about the
kernel at all — they were about my harness. Ring capacity mattered
because my producer and consumer were separate processes. Zero-copy
looked like an 18× win because my destination was `/dev/null`. Both are
properties of the measurement, and I attributed both to the code. The
tell, in hindsight, is that I never varied them: over two days the folio
size, the patch and the pipe size all got swept, some of them across
sixty-four randomised kernel builds, while "two processes" and "discard
the output" never moved once — not because I had decided they were
fixed, but because I had never noticed they were choices.

The patch is not upstream and now probably never will be, at least in
the form I wrote it. That is a fine outcome. The finding underneath it
still states in one sentence — *splice hands a pipe one buffer per page
even when the page cache is handing splice something 512 times larger* —
and the more useful finding, the one I did not go looking for, states in
one too: **most of what people quote about zero-copy is measured against
a destination that does not exist.**
