+++
title = "The mode checker reviewed my code"
date = 2026-07-16
description = "OxCaml's mode system found the one global that made my shrink engine unsafe to parallelise, explained itself, and the forced refactor paid for itself the same afternoon."
+++

My property-testing engine for base_quickcheck had a global. I knew it
was there. It was the convenient kind of global: a single
`Tape.t option ref` that the random-state shim consulted on every
draw, so the engine could install a recording tape without threading
it through any signatures. It worked, all my tests passed, and I had
already written the comment apologising for it.

Then I wanted parallel shrinking. Shrink attempts are embarrassingly
parallel in a choice-tape engine: each attempt replays an edited tape
through the generator, needing nothing from any other attempt. The
obvious question for OxCaml was whether the mode system would let me
say that. So I asked it directly, by claiming that installing a tape
was portable, the mode meaning roughly "safe to use from another
domain":

```ocaml
let probe : (unit -> unit) Modes.Portable.t =
  { portable = (fun () -> Splittable_random.For_tape.set_tape None) }
```

The compiler said no, and it said no with a paper trail:

```
Error: The value "Splittable_random.For_tape.set_tape" is "nonportable"
       but is expected to be "portable"
         because it is used inside the function at line 7
         which is expected to be "portable"
         because it is the field "portable" (with some modality) of
         the record at line 7.
```

No annotations anywhere in my code. The checker inferred, from the
definition alone, that a function closing over a global mutable ref
cannot be handed to another domain, and told me which value, in which
closure, violated which expectation, and why the expectation existed.
This is the code review comment a careful colleague would leave:
"this global will bite you the moment you go parallel", except it is
machine-checked and cannot be argued with.

The fix the checker forces is the design a reviewer would have asked
for anyway: carry the tape inside the random state.

```ocaml
type t =
  { real : Sr_real.t
  ; tape : Tape.t option
  }

let attach t tape = { t with tape = Some tape }
```

No global, no install/uninstall dance, and every shrink attempt
becomes self-contained: its own tape, its own RNG state, nothing
shared. After the refactor the same probe, now building all of its
state inside the closure, compiles and runs. The before and after are
one commit apart, and the diff is the honest one: signatures now say
what the code always meant.

## The compiler had a second opinion

With attempts self-contained I reached for `Domain.spawn` to build a
worker pool, and OxCaml's standard library had opinions about that
too:

```
Alert do_not_spawn_domains: Stdlib.Domain.spawn
User programs should never spawn domains. [...] spawning more than
[recommended_domain_count] domains will significantly degrade GC
performance.

Alert unsafe_multidomain: Stdlib.Domain.spawn
Use [Domain.Safe.spawn].
```

`Domain.Safe.spawn` is the mode-checked variant: it demands a portable
closure, exactly the property the probe above established. The system
composes: the mode error pushed the state into the right place, and
the safe-spawn API is the payoff for having done it, a spawn that the
compiler can check does not smuggle shared mutable state across
domains. (My pool currently uses plain `Stdlib.Domain` so the same
source builds on stock OCaml; switching the OxCaml build to
`Domain.Safe` is the natural next step, and the alert will keep
nagging until I do.)

I want to dwell on what did NOT happen. I did not add mode annotations
to my engine. I did not port anything to a new concurrency framework.
I compiled existing code under a compiler with a stricter type system
and asked it one question, and it found the one piece of state that
made parallelism unsafe, explained itself, and pointed at the
sanctioned alternative. The cost of admission was zero; the review
was free.

## Was it worth it? The numbers

The pool evaluates generation cases (find the failing example) and
deletion-scan proposals (shrink it) in parallel batches. Two details
matter for correctness: taking the lowest failing index in a
generation batch reproduces the sequential engine's choice of failing
example exactly, and shrink acceptance still goes through one
shortlex comparison against the incumbent, so results stay
deterministic. Attempt counts are identical at every domain count.

Benchmark: a bind-heavy generator (length 1..256, then that many
bounded ints), a test body with a fixed ~100us of work, failures in
roughly one case in thirty, twenty trials. First on stock OCaml 5.3:

```
domains= 1  wall 9.16s
domains= 4  wall 2.98s
domains= 8  wall 2.01s
domains=16  wall 1.92s
```

4.6x at 16 domains, saturating because shrinking's bisection passes
are still sequential (Amdahl, as always). And the same binary-except-
compiler under OxCaml:

```
domains= 1  wall 8.03s
domains= 8  wall 2.00s
```

Flambda2 runs the sequential engine 12 percent faster out of the box,
and the parallel ceiling is the same. Identical results throughout:
same failures found, same 57000 shrink attempts, on both compilers at
every width.

## The takeaway

The pitch for OxCaml's modes is usually written from the perspective
of people building concurrent systems on purpose. My engine is a
humbler data point: a small library that was not written with
parallelism in mind, whose author knew about a shortcut and took it.
The mode checker found the shortcut from outside, produced the
refactor a good reviewer would have demanded, and the refactor paid
for itself the same afternoon with a 4.6x wall-clock win.

## Postscript: the checker reviewed the upstream PR too

After this post was drafted, I turned the shim into a proposed
upstream patch: an [Intercept record carried inside
splittable_random's state](https://github.com/janestreet/splittable_random/pull/2),
so any engine can hook the draws. The mode checker had one more
review comment: a state carrying hook closures is nonportable DATA, so
my portable-attempt probe stopped typechecking the moment hooks moved
into the state. It is right again: hooks capture arbitrary state, and
whether the intercept field should demand portable hooks (data-race
freedom composes, but recording observers inherently capture mutable
state) or carry a nonportable modality is a real design decision. It
is the first question in the upstream proposal, and I find it
delightful that the compiler raised it before any human reviewer saw
the patch.

The engine itself, and the shrinking results that motivated all of
this, are the subject of [the previous
post](@/posts/choice-tapes.md). Code at
[github.com/matthiasgoergens/tapecheck](https://github.com/matthiasgoergens/tapecheck).
