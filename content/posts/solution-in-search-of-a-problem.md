+++
title = "A solution in search of a problem"
date = 2026-07-16T23:29:00+08:00
description = "Mozak's zkVM let mutually distrusting programs coordinate without seeing each other's code. I spent months failing to explain why anyone would want that — because most of the field never enters the regime where the problem exists."
+++

A few years ago I was CTO of Mozak, and we built a RISC-V zkVM: write
Rust (or anything that compiles to RISC-V), run it, get a
zero-knowledge proof that it ran correctly. The company is dormant
now, and some of our best design ideas were never written up. This
post starts paying down that debt with the one I had the most trouble
explaining while it mattered: how independent programs coordinate.

The trouble was not that people found the design wrong. It was worse:
I repeatedly failed to convince smart people in the field that it
solved a problem anyone had. At some point I started calling it "a
solution in search of a problem" myself. I now think I know exactly
why the conversations went that way, and the diagnosis is the most
interesting part, so this post tells the story in that order: the
idea, the failure to sell it, the diagnosis.

## Programs that cannot speak

Strictly speaking, a zero-knowledge proof cannot produce output. It
can only accept or reject input. In a non-deterministic setting that
turns out to be enough: instead of *emitting* a value, a program is
handed a candidate value as advice and checks "yes — this is what I
would have said" (or fails). Output becomes validated input. Every
zkVM leans on this trick somewhere.

Our idea was to play the same trick *between* programs. Present every
program participating in a joint action — a token transfer, say —
with a shared **script**: an opaque record of the intended dialogue.
Who called whom, with what arguments, what came back. A program
accepting the script means:

> If the others say what the script says they say, then I approve of
> what the script says I do.

Like two parties signing a contract a lawyer drafted for them, or
diplomats initialling a protocol an aide prepared. Each signs only for
their own part; none needs to read the others' private reasoning. No
call ever happens between programs. Each performs its part alone, in
its own proof, and the parts line up because each was validated
against the same script.

To the proof system the script is a black box of bytes — its meaning
is whatever convention the programs share. One thing, though, is
verified rather than conventional: the **cast**, the ordered list of
programs party to the script, each named by its code hash. The
verifier checks each proof, checks every seat in the cast is matched
by a proof of exactly that program at exactly that position, and
checks everyone was bound to the same script. It never reads what
anyone said.

So: a token program insists the spending wallet approves of each
transfer; a wallet approves by proving its owner signed
off; each is a black box to the other; and a transaction settles when
all the proofs land and the cast checks out.

## "Why not just verify the proof?"

Whenever I pitched this, the counter-proposal was always the same,
and always reasonable: have the token program *verify the wallet's
proof* inside its own execution. Proof recursion. Every major zkVM
supports it, and it is the answer the field reaches for by reflex.

Recursion is sound, and for its intended job — compressing many
proofs into one — we used it too. But as a *coordination* mechanism
it has a closed-world assumption baked in: to verify the wallet's
proof, the token program must bake the wallet's circuit into its own *at
authoring time*. Now run the scenario that actually motivated us. A
corporate wallet requires CEO and CTO approval, or a sign-off from
compliance above some threshold — private, changing rules. Years
after the token shipped, a regulator requires a screening attestation
from an accredited provider for large transfers. Under recursion the
token program would have needed the compliance provider's
circuit baked in before that provider existed. Nobody can be
added to a transaction who was not foreseen by the programs already
in it, and some program must sit at the root of the verification tree
and see everyone.

In the script model the compliance program just joins the cast. The
wallet's policy demands its presence; the token program neither knows
nor cares — with separate scripts, a composition mechanism the later
posts have to make precise, it need not even learn compliance was
involved. The participant set is decided per transaction, not
frozen per program.

I found this argument completely convincing. The people I pitched it
to mostly did not. They kept asking, gently, when a set of programs
that genuinely do not trust each other, cannot read each other, and
were written in ignorance of each other would ever need to agree on
one atomic action. Fair question. Here is the answer I wish I had
had.

## The one-program world

Almost everyone building zk systems is, at the proof level, running
exactly one program: an interpreter. On Ethereum rollups it is the
EVM; elsewhere it is some other state-transition function. The
contracts people actually write live one storey up, as *data* that
the interpreter executes. And at that level, contracts are not black
boxes at all. The interpreter holds all their bytecode, executes
every one of them, mediates every call, owns all the shared state.
Cross-program coordination is solved by ordinary interpreter
mechanics — transparently, by a single executor that sees everything.
The proof at the bottom certifies one sentence: *the interpreter ran
correctly*.

If that is your world, my problem statement is unintelligible.
Mutually distrusting programs? They all trust the interpreter.
Opaque counterparties? The executor reads every contract. Unforeseen
third parties? Deploy another contract; the interpreter mediates. The
problem does not exist — because the architecture bought its absence,
and paid with a single all-seeing executor, contracts that are public
to it, one monolithic execution trace, and the standing cost of
proving an interpreter interpreting instead of programs running.

Mozak made the opposite choice: no interpreter. Contracts are
zkVM-native programs, each proven directly, by whoever wants to prove
it. The moment you make that choice you *have no executor to mediate
calls* — so you are forced to invent coordination that isolated,
independently-generated proofs can each check alone. That is the
script. The design was never a feature we bolted on; it is what
remains of "A calls B" after you delete the layer that was doing the
calling.

Distributed systems has a name for this distinction. *Orchestration*:
a central conductor holds the logic and tells every participant what
to do — the EVM, exactly. *Choreography*: no conductor; each
participant knows its own part and stays in step by following a plan
agreed in advance. Programs on Mozak are choreographed. (The theatre
metaphor runs deeper — the participant list was literally called the
`cast_list` in our code — and the precise mechanics of scripts,
seats, and settlement deserve their own post.)

## What refusing the interpreter buys

Everything follows from the one move. Nobody holds all the programs,
so different distrusting parties prove different programs separately
— a hardware signer can produce the wallet's approval offline on
Monday and a relayer can assemble the settlement on Friday without
ever touching a key. Nobody *sees* all the programs, so counterparties
stay opaque: in the finished design, a program reveals neither its code
nor its private inputs, only what the script says. (The prototype
proved execution; the zero-knowledge blinding itself was never built.)
Mediation lives in a document
rather than an executor, so the participant set stays open — the
compliance program nobody foresaw joins by appearing in a cast. And
there is no interpreter in the circuit to pay for: programs compile
to the machine's own instruction set and are proven directly.

The honest comparison, briefly. Recursion-as-coordination assumes the
relationships are known at authoring time — precisely when you do not
have this problem. Monolithic proving (zkEVMs, Cairo) assumes a single,
all-seeing executor and public contracts. The nearest cousin is Aleo,
which also proves calls as separate transitions bound into a
transaction — and where, as [Equilibrium's deep
dive](https://equilibrium.co/writing/privacy-blockchains-and-aleo-deep-dive)
puts it plainly, "there is no composability between private
applications": one application cannot take another's private state as
input. That wall is exactly where the script begins.

## Where this is going

The regime where all the assumptions fail together — mutual
distrust, mutual opacity, participants unknown in advance — is not
exotic. It is regulated custody, cross-institution settlement,
confidential governance rules, private applications that need each
other's verdicts. The field does not lack the problem; it lacks it
*visibly*, because the dominant architecture dissolves it at the cost
of an all-seeing middleman, and everyone has stopped seeing the cost.

That is the diagnosis, and also the apology I owe my past
conversation partners: I was describing a solution to a problem my
listeners' architecture had defined away.

I am resurrecting these ideas from [the dormant
codebase](https://github.com/0xmozak/mozak-vm), which is frozen, so
the write-ups are landing here instead. The posts that follow will cover the
mechanism in earnest: how settlement binds proofs to seats without
reading anyone's output, why "who are you in this transaction" is a
subtler question than it looks, and how we made global state advance
in parallel by making most operations commute. The prototype was
never finished. The ideas, I think, deserve to be.

That last sentence is also an invitation. Finishing them no longer
means building a zkVM: the 2026 provers supply execution proofs and
recursion off the shelf, so what remains is the part that was always
the point — the coordination layer, the state model, and a handful of
open problems the next posts will state precisely. If picking up a
good unfinished idea appeals to you, [write to
me](mailto:matthias.goergens@gmail.com).
