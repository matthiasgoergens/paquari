+++
title = "Smashing the Bitcoin Piñata"
date = 2026-08-12T13:00:00+08:00
description = "I spent several days auditing a 2017 OCaml TLS stack that protected $100,000 in Bitcoin for three years. The code held up better than I expected."
+++

The Bitcoin Piñata was a MirageOS unikernel that held the private key to a
Bitcoin address containing 10 BTC. You could connect to it over TLS. If you
completed a mutual-authentication handshake — presenting a certificate signed
by the Piñata's own CA and proving you held the corresponding private key —
it would send you the Bitcoin key. The challenge ran from February 2015 to
March 2018 and nobody claimed the prize. The owners eventually withdrew the
funds themselves.

The question that lingered: was the TLS stack actually solid, or did nobody
look hard enough?

I spent several days in August 2026 finding out. I pointed two different model
families at the code — Codex (GPT-based) and DeepSeek — and asked each to find
every vulnerability it could. Then I built a Docker lab reproducing the exact
2017 environment and verified every claim that could be tested against a
running unikernel. Here is what I learnt.

## The target

The Piñata ran a pinned stack from the [mirleft](https://github.com/mirleft)
project, which set out to build a "not quite so broken" TLS implementation in
OCaml: `ocaml-tls` 0.9.0, `ocaml-x509` 0.6.0, `nocrypto` 0.5.4, all from
December 2017 or January 2018. Five ports:

- **:80** — an HTTP page explaining the challenge, CA certificate included
- **:443** — HTTPS with a server cert signed by the Piñata CA, no client auth
- **:10000** — mutual TLS: the Piñata is the server, you must present a valid
  client certificate, it sends the Bitcoin key on success
- **:10001** — callback: you connect via TCP, the Piñata hangs up, then
  connects back to your IP on port 40001 as a TLS client, validates your
  server certificate, sends the key
- **:10002** — reverse: you connect via TCP, the Piñata initiates TLS as a
  client over that channel, validates your server certificate, sends the key

On the three authenticated ports the Piñata generates a fresh 4096-bit RSA CA
at every boot and uses it as the sole trust anchor. To extract the secret you
need a certificate signed by that CA *and* the corresponding private key to
produce a CertificateVerify signature over the handshake transcript. The web
page puts it plainly: "No, you can't have the certificate key."

## How I approached it

Rather than read the code line by line, I ran two adversarial audits in
parallel. The TLS layer went to one model, the X.509 layer to another. Each
was told to *break* the code, not to summarise it. Disagreement between the
two families is a finding worth resolving by measurement; agreement is the
strongest cheap evidence available.

The TLS audit produced eight ranked candidates. The X.509 audit produced ten.
Both are in the [lab repository](https://github.com/matthiasgoergens/btc-pinata-lab),
together with a Docker image that reproduces the exact Piñata environment:
Debian 9, OCaml 4.06.0, every opam package pinned to its January 2018 version.
Every claim that could be tested against a running unikernel was tested.

## What I found

### The eight-year-old five-byte crash

The most striking bug was in `parse_change_cipher_spec`, the TLS record parser
for the ChangeCipherSpec message. It was the only wire parser in the entire
library not wrapped in the `catch` combinator that converts exceptions into
handled errors:

```ocaml
(* lib/reader.ml:118-121 — the ONLY parser without `catch` *)
let parse_change_cipher_spec buf =
  match len buf, get_uint8 buf 0 with   (* get_uint8 on empty buf *)
  | 1, 1 -> return ()
  | _    -> fail (Unknown "bad change cipher spec message")
```

OCaml evaluates both expressions in the match tuple before matching.
`get_uint8 buf 0` on an empty buffer raises `Invalid_argument` — cstruct 3.2.1
bounds-checks the access — and the exception propagates through the TLS stack
uncaught. Every other parser catches this through the `catch` wrapper, which
converts `Invalid_argument` into a handled `Underflow` error. This one does
not.

Five bytes from netcat — `14 03 03 00 00`, an empty ChangeCipherSpec record —
trigger it on any TLS port, pre-authentication. The commit that fixed it, on
2026-03-09 (version 2.0.4), reads: "present since the initial release 0.1.0."
Eight years and three months.

In practice the exception is caught by Lwt at the monad boundary and kills
only the connection, not the whole unikernel. Still, five bytes, no
authentication, any port, eight years.

### Memory exhaustion through unbounded fragments

Handshake messages in TLS can be up to 2^24−1 bytes, about 16 MiB. The Piñata
buffers incomplete messages in `hs_fragment` with no size cap, and the
accumulation happens in `handle_packet` before any state-machine check — you
do not need to complete a handshake, or even start one properly. One connection
can pin up to 16 MiB by declaring a 16 MiB Certificate message and
slow-dripping bytes. Ten connections with a single 16 KiB fragment each
increased the unikernel's RSS from 39 MiB to 56 MiB in my lab.

Fixed in version 0.12.0 (May 2020), though the fix was a side-effect of a TLS
1.3 refactor rather than a direct response to the issue.

### Exponential path-building denial of service

The X.509 validator's `build_paths` function builds *all* DN-consistent paths
through the presented certificate chain before checking any signatures. With
N certificates all sharing the same subject and issuer distinguished name, it
enumerates O(N!) paths — each one a list of certificate records allocated and
traversed.

I generated eleven self-signed certificates with matching DNs, sent them as a
client certificate chain to port 10000, and watched the unikernel hang for over
thirty seconds. More certificates would wedge it indefinitely.
Pre-authentication. Triggers on any port that validates a certificate chain.
Still present in current upstream, though partly mitigated by the fact that
real certificate chains are short.

### Missing KeyUsage and ExtendedKeyUsage checks (CVE-2026-45389)

The server-side client-certificate validation checked that the chain was signed
by the CA and that the RSA key was at least 2048 bits — and nothing else. It
did not verify that the leaf certificate's `KeyUsage` included
`digitalSignature` or that its `ExtendedKeyUsage` included `clientAuth`.

The *client* side of the same library *did* check EKU for server certificates.
The error constructors `InvalidCertificateUsage` and
`InvalidCertificateExtendedUsage` existed in the code but were only wired up on
the client path. The asymmetry is conspicuous once you see it.

This is a force-multiplier, not a standalone break: if any chain-validation
bypass lets you present a wrong-purpose certificate as a client certificate
— say, the Piñata's own public web server certificate, which anyone can harvest
by connecting to port 443 — the missing KU/EKU check means no second line of
defence catches it. You still need the private key for CertificateVerify, so
the barrier holds, but one fewer thing stands between a chain bug and a total
win.

Fixed in version 2.1.0 on 2026-05-19, reported by Ben Smyth.

### The Piñata trusts itself

Because all three authenticated ports share a single CA, the Piñata's own
endpoints can authenticate each other. Bridge port 10000 — which needs a
client certificate — to port 10002 — where the Piñata *is* a TLS client
presenting its own client certificate — with a TCP relay, and the mutual-TLS
handshake completes. The secret flows in both directions as encrypted
application data. The same trick works through the port 10001 callback
mechanism by relaying port 40001 back to port 10000.

This is not a bug. The web page even invites you to "enjoy watching it do so."
But it is a reminder that the Piñata's security is one-deep: compromise or
confuse any single endpoint, and the secret leaks.

## What I did not find

No chain-validation bypass. Every path from a presented certificate to the
trust anchor ends in a strict RSA-PKCS#1-v1.5 signature verification against
the 4096-bit CA key. The parser enforces outer-signature-algorithm equals
inner-TBS-signature-algorithm, requires strict DigestInfo parsing with a
no-leftovers check, and does full-length hash comparison. The `raw_cert_hack`
for TBS extraction — a hardcoded 15-byte offset arithmetic — is ugly but fails
closed: the strict parser only admits RSA signature algorithms with NULL
parameters, which are exactly 15 bytes, and ECDSA issuers are rejected anyway.

No certificate-verify bypass. The server binds `session.peer_certificate` to
the head of the validated chain, and CertificateVerify is verified against
exactly that certificate's key over the full handshake transcript — ClientHello
through ClientKeyExchange. The TLS 1.2 hash must be in the server's configured
hash list, which runs SHA-512 through SHA-1 with MD5 excluded. There is no
path to complete the handshake on port 10000 without the leaf private key.

No RNG attack. The CA key is generated by `Nocrypto.Rsa.generate` at boot using
the Fortuna PRNG seeded from `/dev/urandom`. Two boots produced different keys.

The Marvin-class timing side-channel on port 443 is theoretically present — the
RSA decryption uses Zarith `powm` with value-dependent timing, and the blinding
nonce adds per-operation jitter that only averages out over many samples — but
exploiting it would require a research-grade measurement campaign against a
4096-bit key that protects nothing but a static web page. The secret-bearing
ports require client authentication before the RSA decryption step, so the
oracle is not reachable there.

## Reproducing the findings

Everything is in [the lab repository](https://github.com/matthiasgoergens/btc-pinata-lab).

```sh
cd lab
./build.sh                           # Docker image (~5 min first time)
./scripts/run-lab.sh "my secret"     # boot unikernel, run legit test + short-circuit
./scripts/verify.sh                  # run all five vulnerability checks
```

The Docker image is about 3 GB. Rebuilds take about 30 seconds — the opam
layer is heavily cached.

To try the five-byte crash yourself while the unikernel is running:

```sh
printf '\x14\x03\x03\x00\x00' | nc 127.0.0.1 443
```

The connection dies. The unikernel stays up. Eight years, five bytes, no
authentication.

## What this says about the stack

The `mirleft/ocaml-tls` authors set out to build a TLS stack that was "not
quite so broken." The Piñata was their public bet that they had succeeded.
After two adversarial audits by independent model families and empirical
verification of every actionable finding, the core claim holds up: there is
no chain-validation bypass, no certificate-verify bypass, no way to extract
the secret without either the CA private key or a 4096-bit RSA forgery.

The bugs that *were* found are real — an eight-year-old crash bug, an
exponential DoS, a missing-purpose-check that shipped as a CVE — and they
matter. But they sit at the perimeter, not at the core. The mutual-TLS
handshake does what it says on the tin.

For a roughly 5,000-line TLS library written in a memory-safe language by a
small team, that is a remarkable result. The formal-methods community
sometimes talks about software that does not have bugs, only security
properties that have not been proved yet. The Piñata's TLS stack was not
formally verified, but it was *audited* — adversarially, by multiple
independent reviewers, against a concrete security goal with a known bounty —
and it held.

The 10 BTC are gone, withdrawn by the owners in 2018. But the question the
Piñata asked — can you build a TLS stack that is not quite so broken? — got
a better answer in 2026 than it had in 2018.
