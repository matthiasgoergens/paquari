+++
title = "The two-time pad wanted a 5-gram, not a neural net"
date = 2026-07-16T23:28:00+08:00
description = "For years I threw LSTMs, unsupervised diff-recovery, and a 15 GB corpus at a reused one-time pad. What actually broke it was a character 5-gram and a beam search that refuses to commit early — and I should have tried that on day one."
+++

*The original puzzle, preserved on the Internet Archive: [Decrypting the Two-Time Pad](https://web.archive.org/web/20070303124822/http://www.itasoftware.com/careers/puzzle_archive.html?catid=39) — scroll down past the other puzzles to find it.*

There is an old ITA Software hiring puzzle from around 2004: you intercept two
messages, both encrypted with a one-time pad over a 46-character alphabet
(space, `A–Z`, `0–9`, and nine punctuation marks). Encryption is character-wise
addition modulo 46. A one-time pad is the one cipher that is provably
unbreakable — given a single ciphertext, any plaintext of the right length
could have produced it, and no amount of computation tells you which. The catch,
and the whole puzzle, is in the word *one*. The sender used the **same** pad for
both messages.

That single reuse is fatal, and the reason is a one-line observation. Write the
ciphertexts as `C1 = P1 + K` and `C2 = P2 + K`, all mod 46. Subtract them, and
the pad — the same random `K` in both — cancels:

```
C1 − C2 = (P1 + K) − (P2 + K) = P1 − P2   (mod 46)
```

The randomness is gone. What is left is the character-wise difference of the two
*plaintexts*, known exactly, at every position. The cryptography has evaporated
and left behind a language puzzle: given `P1 − P2`, find two English texts that
have that difference.

I understood this part on the first afternoon. Everything I did *after* it is the
story, and most of it was a mistake.

## The descent

The obvious thing, if you have spent any time around machine learning, is to
learn the map directly. Generate endless training pairs from known books, feed a
network the differences, train it to emit the two plaintexts. It barely learned.
For a long time I assumed I was holding it wrong — wrong architecture, wrong
loss, not enough data — when in fact the framing was doomed. The map from a
window of differences to a plaintext character is savagely multimodal: many
locally-plausible splittings agree with the same differences, and which one is
right depends on context that can sit dozens of characters away. Train a network
to regress on that with hard labels and it dutifully averages all the valid
answers into mush. Nothing local and feed-forward can do this, because the
disambiguating evidence is not local.

So I concluded I needed a *real* language model, and went to build one. This is
where the git history gets embarrassing. I did an architecture search: dense
residual networks with PReLU, GRUs, a plain RNN, DenseNet-style skip connections,
before settling on a two-layer LSTM — BatchNorm before the recurrence, mixed
precision, a 150-character window, gradient clipping, learning-rate schedules. I
preprocessed 15 GB of Project Gutenberg into byte indices with a Rust tool that
memory-mapped the corpus and streamed random snippets to the GPU over a pipe, so
I would never run out of training data. There are branches in that repository
called `fractal`, `frac-again`, `truncated`, and `more-loss`. There is a whole
line of work on recovering language statistics from the differences *alone*,
never looking at a plaintext — minimum-entropy objectives, iterative proportional
fitting, and eventually an FFT-based gradient-descent scheme for trigrams that I
was very proud of for getting a "77.6% improvement" on a number that should never
have been on the critical path.

All of it was in service of a prior. And the prior was not the hard part.

## The part that mattered

The thing that actually broke the cipher is a beam search over the two texts at
once. Walk left to right. At each position, for each surviving hypothesis, try
all 46 possible characters for `P1`; the corresponding `P2` character is *forced*,
because their difference is known. Score how English-like both halves are under
your language model, keep the best few thousand hypotheses, prune the rest. At
the end, read off the lowest-loss pair.

Once that was in place, it worked — and the model barely mattered. The LSTM was
enormous overkill for the job it was doing. Which raises the obvious question I
had spent two years not asking: how well does the *dumbest possible* prior do?

## The baseline I skipped

So I rebuilt the whole thing from scratch around the baseline I should have
started with. The language model is a character 5-gram: for every four-character
context, count what character came next, across thirty-five public-domain
books. Unseen contexts are interpolated with shorter ones, Witten-Bell style, so
nothing ever gets zero probability. That is the entire model. It has no
parameters to train, no GPU, no schedule. Built from 24 million characters, it
takes about seven seconds.

Drop it into the same beam search, and it recovers about **97% of the characters**
of nineteenth-century prose (96.8% on a broader held-out mix). The errors cluster
on proper nouns and digit runs — the
places where both texts are genuinely unpredictable at the same spot, and the
information to separate them simply is not there. And here is the part that
stung: widening the beam past about four thousand hypotheses changed the answer
by nothing at all. The search was already sitting at the model's ceiling. Every
gigabyte of corpus and every GPU-hour of LSTM had been buying me almost nothing
that seven seconds of counting did not already provide.

Pointed at the puzzle's actual ciphertexts, it resolves them into the opening of
Thomas Hardy's *The Mayor of Casterbridge* — the hay-trusser walking to
Weydon-Priors with his wife and Elizabeth-Jane — and a chapter of Darwin's
*Voyage of the Beagle*, the one about the volcanic craters and giant tortoises of
the Galápagos. Neither book is in the training corpus. You can paste the
recovered text into a search engine and name them both.

## The lesson I would actually keep

It would be cheap, and wrong, to conclude that n-grams beat neural networks. They
do not, in general. The real lesson is about what a baseline *buys* you, and it is
not just that it is cheaper.

This problem has two independent hard parts: is the search right, and is the
prior good enough? Jumping straight to the LSTM entangled them. When nothing
worked, I could not tell whether the beam was broken, or the model undertrained,
or the optimiser merely sulking — three sources of failure firing at once, which
is exactly the fog that turns into years of rabbit holes. A 5-gram nails "good
enough prior" in seven seconds and lets you test the search *in isolation*. And
the search was the entire discovery. I had it early, and then buried it under a
model for two years.

The reframe that makes this obvious in hindsight is that the language model —
neural or counted — is only ever a *smoother*: an estimate of `P(next char |
context)`. The beam search is identical whatever you plug in. My LSTM was an
extravagantly expensive way to answer a question a lookup table answers well
enough. And the deepest reason the puzzle is solvable at all is the same fact
from the other side: a one-time pad is safe only because the pad looks like pure
noise and demands the message look equally like noise. English is the opposite of
noise. Reusing the pad leaks exactly the redundancy that lets you pull two
overlaid texts back apart — and that redundancy is so thick that counting
letters, and refusing to commit too early, is enough.
