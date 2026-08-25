+++
title = "A proof that 0 = 1, in a real zk-VM"
date = 2026-07-16T23:27:00+08:00
description = "Two syntax-tree nodes in Polygon Miden hashed to the same value, a gap the team had already flagged in a TODO. I turned it into a running proof that a program outputting 0 outputs 1 instead."
+++

A zero-knowledge VM makes one promise to a verifier: *I ran the program
whose hash is H, on this input, and got this output, and here is a
proof you can check without re-running anything.* The whole edifice
rests on it being computationally infeasible to find a different
program with the same hash. In December 2022 I built a Miden Assembly
program that outputs a stack of zeros, together with a valid proof that
the same hash identifies a program outputting one. Both claims verify.
A proof that 0 = 1.

The Miden team had already recorded the underlying weakness in a `TODO`
comment. I recognised it as an exploitable soundness hole rather than a
tidiness issue and built a working exploit.

Here is the program:

```
begin
  if.true
    push.1
  end
end
```

Run it and you get zeros. [The
proof](https://github.com/matthiasgoergens/miden-collision) says you
get a one. Neither the executor nor the verifier is buggy in
isolation; they simply disagree about which program the hash refers
to, and that gap is the whole exploit.

## Programs are trees, and the tree is the identity

Miden does not hash the text of your assembly. It compiles the program
to a Merkleised abstract syntax tree (a MAST), and the program's
identity is the hash of the tree's root. Two node types matter here:

- `SPLIT(a, b)` is a conditional. It reads the top of the stack; if it
  is one, it runs `a`, and if it is zero, it runs `b`. The `if.true`
  above compiles to a `SPLIT` whose true branch is `push.1` and whose
  false branch is an empty (noop) span. On the default stack of
  zeros, `SPLIT` takes
  the false branch and does nothing. Output: zeros.
- `JOIN(a, b)` is a sequence. It runs `a`, then runs `b`,
  unconditionally.

Now the flaw. A `SPLIT` node and a `JOIN` node with the same two
children hashed to the *same value*. The hash was
`Hash(child_a, child_b)` with nothing to say which node type it was.
This was not hidden. The comment above the `Join` block's hashing code
said so in as many words:

```rust
/// Hash of a Join block is computed by hashing a concatenation of the
/// hashes of joined blocks.
/// TODO: update hashing methodology to make it different from Split block.
```

A `TODO` is a note to future maintainers that a thing is unfinished. It
is easy to read this one as a matter of neatness. What it actually
describes is that two different programs share a hash, and in a system
whose entire security argument is "the hash identifies the program",
that is soundness. So these two trees are indistinguishable by hash:

```
SPLIT(SPAN push.1, SPAN noop)      # the honest if.true, outputs 0
JOIN(SPAN push.1, SPAN noop)       # runs push.1 then noop, outputs 1
```

The `JOIN` version runs `push.1` and then the no-op, and leaves a one
on top of the stack. It has the same hash as the honest conditional.
So I proved the execution of the `JOIN` tree, honestly, and handed the
verifier a proof whose program hash matches the `SPLIT` tree that any
normal user would have written. The verifier checks the proof, checks
that the hash is the one it expected, and concludes that the
zero-outputting program output a one.

The one wrinkle was construction. The Miden Assembly compiler never
emits the `JOIN` variant, because `if.true` always lowers to `SPLIT`.
So there is no source program that produces the malicious tree; I had
to build the MAST by hand, which meant hacking up the compiler to let
me assemble the collided tree directly. The hack was not pretty, and I
said so in the report.

## Reporting it in the zero-knowledge spirit

I filed the issue as a teaser: here is a program that outputs zeros,
here is a proof it outputs a one, the proof is in this repo, have fun.
No explanation of the mechanism. It seemed only fitting to report a
zero-knowledge bug with a zero-knowledge disclosure, and since the team
already knew about the underlying `TODO`, the fun part was watching
them connect it to a concrete exploit.

Bobbin Threadbare, who leads Miden, did exactly that within a day:
"my guess is that you constructed two Miden VM programs which do
different things but hash to the same value. This is actually not too
difficult to do right now as `JOIN` and `SPLIT` blocks are hashed in
exactly the same way," with a link straight to
the `TODO`. Exactly right. I confirmed it, and added the thing that
worried me more than this single instance: the code kept picturing an
attacker who crafts malicious *source programs*, when a real attacker
works one level down, at the tree the source compiles to. Wherever
that assumption hid, there were probably more collisions like this one.
The team agreed, and treated the whole thing as the fun collaborative
puzzle it was.

## Fixing a collision without paying for it

Different node types needed different hashes, but adding circuit operations
would increase the cost of every instruction, and the hash function had no
unused input slots.

The thread turned into a small design discussion of the options, and
Edward Kmett showed up with the cleanest of the arithmetic ones. Rather
than widening the hash to take an extra "which node type" input (there
were no free slots), apply a different affine transformation to one of
the hash inputs per node type: hash `3a + 1, b` for one kind and
`5a + 2, b` for another. Distinct linear adjustments send otherwise-equal inputs to unrelated
outputs, so all tree shapes hash apart, at a cost of one multiply and
one add per node against the full price of a
cryptographic permutation. He preferred an affine map over a bare
multiply precisely because one of the real hash inputs was zero, and a
constant times zero is still zero. He even put a fix over the fence as
a PR, mostly to unblock his own team while the discussion continued.

The other candidates each had a catch at the time. Domain separation,
setting a reserved capacity register of the sponge to a per-node-type
value, is the simplest and cheapest idea — and was the Miden team's
preferred direction — but the sponge's capacity is also its security
margin, and spending an element of it was believed to cost security
bits. Using different output slots as the digest costs no field
operations but complicates the constraint system.

Miden's concurrent migration to a new hash function, RPO (Rescue-Prime
Optimized), changed the arithmetic of the decision. RPO's redesigned padding
scheme
freed the second capacity element, and its security analysis showed
that spending it costs nothing below the 128-bit security target, so
the objection to domain separation, the one thing that had made it look
expensive, simply evaporated. [The fix they
shipped](https://github.com/0xMiden/miden-vm/pull/682) is not Kmett's
affine transform; it writes a domain specifier into the second capacity
element when hashing a control block, composed from the block's opcode
bits.

I still find the affine mixer the prettier answer, for a reason that
outlives this particular bug. If your hash makes it infeasible to find
attainable digests at a chosen affine relation, linearly adjusted inputs
behave like unrelated inputs, so a distinct multiplier per node type
gives you a large practical namespace at the price of one multiply and
add and no extra state. A capacity register works differently, but it
does not spend a bit or a field element for each domain: every node type
can share one register and select a different tag value. Its namespace
is limited by the tag encoding, not by a security budget consumed one
domain at a time. The real trade-off is whether dedicating that part of
the sponge state to a tag leaves enough capacity for the target security
level; for RPO, the analysis said that it did.

RPO gave Miden a free register, making domain separation the cheapest option
in their actual constraint system even though I still prefer the affine mixer
in isolation.

## Why this genre of bug is worth staring at

The executor, verifier, and hash function each behaved as specified, but the
executor and verifier assigned different meanings to the same hash. The system
as a whole would therefore certify that zero equals one.

That is the shape of most soundness bugs I have seen in
proof systems, and it is why they reward a particular habit: stop
reading the source language and start reading the object the source
compiles to, the thing the proof actually talks about. The MASM
programmer cannot write the malicious tree. The person holding the
MAST can. A zero-knowledge proof is only as honest as the computational
binding between a program and its hash. An ambiguous tree encoding can
break that binding with a structural collision hiding in it, even, as
here, one the authors had already flagged.

Miden had documented the known gap and fixed the general encoding problem once
shown a working witness. My contribution was the witness: a running artefact
that demonstrated the practical consequence of the `TODO`.

The full artefact, including the Dockerised proof (runnable as of
December 2022 — the script tracks Miden's live `next` branch, which has
long since moved on), is [on GitHub](https://github.com/matthiasgoergens/miden-collision);
the original report and the design discussion are [Miden VM issue
605](https://github.com/0xMiden/miden-vm/issues/605).
