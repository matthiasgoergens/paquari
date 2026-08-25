+++
title = "A source audit of the Bitcoin Piñata"
date = 2026-08-12T13:00:00+08:00
description = "I spent several days auditing a 2017 OCaml TLS stack that once protected 10 BTC. The code held up better than I expected."
+++

The Bitcoin Piñata was a MirageOS unikernel holding the private key to 10 BTC.
Connect over TLS, complete a mutual-authentication handshake with a certificate
signed by the Piñata's own CA, prove you hold the corresponding private key,
and it sends you the Bitcoin key. The challenge ran from February 2015 to March
2018. Nobody claimed the bounty, and no successful compromise was publicly
reported. The owners withdrew the funds themselves.

Three years, roughly US$200,000 at the December 2017 Bitcoin peak. Was the TLS
stack actually solid, or did nobody look hard enough?

I spent several days in August 2026 finding out. I pointed two different model
families at the code — Codex (GPT-based) and DeepSeek — and asked each to
*break* it. Each model's output served as a hypothesis generator; every finding
was checked against the source by me. Then I built a Docker lab with a
source-compatible reconstruction of the January 2018 package set and verified
each concrete reproducer against a running unikernel.

## The target

The Piñata ran a pinned stack from the [mirleft](https://github.com/mirleft)
project, which set out to build a "not quite so broken" TLS implementation in
OCaml. The opam manifest pins the project's `#pinata` branches —
`ocaml-tls` at a844e52 (one commit past the 0.9.0 release, raising the minimum
DH and RSA sizes to 2048 bits), `ocaml-x509` at 31abe40 (disabling the
`notBefore` check for the Piñata deployment), `nocrypto` 0.5.4 from January
2017, and `asn1-combinators` 0.2.0. Five ports:

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

## What I found

### The nearly-twelve-year-old five-byte crash

The most striking bug was in `parse_change_cipher_spec`, the only wire parser
in the entire library not wrapped in the `catch` combinator that converts
exceptions into handled errors:

```ocaml
(* lib/reader.ml:118-121 *)
let parse_change_cipher_spec buf =
  match len buf, get_uint8 buf 0 with
  | 1, 1 -> return ()
  | _    -> fail (Unknown "bad change cipher spec message")
```

OCaml evaluates both expressions in the match tuple before matching.
`get_uint8 buf 0` on an empty buffer raises `Invalid_argument` — cstruct 3.2.1
bounds-checks the access — and the exception propagates through the TLS stack
uncaught. Every other parser runs through `catch`, which converts
`Invalid_argument` into a handled `Underflow` error. This one does not.

Five bytes — `14 03 03 00 00`, an empty ChangeCipherSpec record — trigger it
on any active ocaml-tls connection, pre-authentication (confirmed on ports
:443 and :10000; port :10001 is a plain-TCP trigger so the initial connection
is not affected). The release notes for version 2.0.4, tagged 2026-03-10,
read: "present since the initial release 0.1.0." Version 0.1.0 is dated July
2014 — nearly twelve years.

In practice Lwt catches the exception at the monad boundary and kills only the
connection, not the whole unikernel. An unauthenticated five-byte record could
therefore terminate a connection in every affected release for nearly twelve
years.

### Memory exhaustion through unbounded fragments

Handshake messages in TLS can be up to 2^24−1 bytes, about 16 MiB. The Piñata
buffers incomplete messages in `hs_fragment` with no size cap, and the
accumulation happens in `handle_packet` before any state-machine check — you
do not need to complete a handshake, or even start one properly. One connection
can pin up to 16 MiB by declaring a 16 MiB Certificate message and
slow-dripping bytes. Ten connections with a single 16 KiB fragment each
increased the unikernel's RSS from 39 MiB to 56 MiB in my lab.

A patch in version 0.12.0 (b17b64f, May 2020) moved the fragment-state update
to occur before processing complete messages — a TLS 1.3-driven refactor — but
it added no length cap and does not invoke the state machine while a declared
16 MiB message remains incomplete. The attack still works against current
upstream (v2.1.2).

### Exponential path-building denial of service

The X.509 validator's `build_paths` function builds *all* DN-consistent paths
through the presented certificate chain before checking any signatures. With
N certificates all sharing the same subject and issuer distinguished name, it
enumerates factorially many paths — each one a list of certificate records
allocated and traversed.

I generated eleven self-signed certificates with matching DNs, sent them as a
client certificate chain to port 10000, and the unikernel stalled for over
thirty seconds. Runtime grows factorially with the number of matching-DN
certificates. Pre-authentication, triggers on any port that validates a
certificate chain. Still present in current upstream (the attacker supplies
the chain, so "real chains are short" does not help).

### Missing KeyUsage and ExtendedKeyUsage checks (CVE-2026-45389)

After ordinary X.509 path validation — certificate parsing, validity checks,
CA-constraint enforcement, chain construction, trust-anchor selection, and
signature verification — the server-side TLS code performed no client-role
KeyUsage or ExtendedKeyUsage check on the leaf certificate. It did not verify
that `KeyUsage` included `digitalSignature` or that `ExtendedKeyUsage`
included `clientAuth`.

The *client* side of the same library *did* check EKU for server certificates.
The error constructors `InvalidCertificateUsage` and
`InvalidCertificateExtendedUsage` existed in the code but were only wired up on
the client path. The asymmetry is conspicuous once you see it.

This is an impersonation vulnerability: an attacker who legitimately owns the
private key for a trusted but wrong-purpose certificate can authenticate as a
client directly. For the Piñata specifically, the ephemeral CA means the only
trusted certificates are the ones the unikernel generates at boot, so the
practical impact is limited — you would still need to obtain a CA-signed
certificate and its key. But in a less constrained deployment the missing
check is a direct impersonation path.

Fixed in version 2.1.0 on 2026-05-19 (release tagged 2026-05-20), reported by
Ben Smyth. The fix treats absent KU/EKU as unconstrained rather than requiring
the extensions to be present.

### The Piñata talks to itself

Because all three authenticated ports share a single CA, the Piñata's own
endpoints authenticate each other. Bridge port 10000 — which needs a client
certificate — to port 10002 — where the Piñata *is* a TLS client presenting
its own — with a TCP relay, and the handshake completes. The secret flows in
both directions as encrypted application data. The same trick works through
the port 10001 callback by relaying port 40001 back to port 10000.

Although the web page invites you to "enjoy watching it do so" and the relay
operator sees only ciphertext, the maintainers' retrospective
describes the same observation: "we can observe the encrypted private key in
transit." The relay demonstrates that the Piñata's security is one-deep: every
endpoint trusts the same CA, so a compromise anywhere breaks the system.

## What I did not find

I found no chain-validation bypass in the examined January 2018 TLS and X.509
code paths. Every presented certificate chain ends in a strict
RSA-PKCS#1-v1.5 signature verification against the 4096-bit CA key. The parser
enforces outer-signature-algorithm equals inner-TBS-signature-algorithm,
requires strict DigestInfo parsing with a no-leftovers check, and does
full-length hash comparison. The `raw_cert_hack` for TBS extraction — a
hardcoded 15-byte arithmetic — is ugly but fails closed: the strict parser only
admits RSA signature algorithms with NULL parameters, which are exactly 15
bytes, and ECDSA issuers are rejected anyway.

I found no certificate-verify bypass. The server binds
`session.peer_certificate` to the head of the validated chain, and
CertificateVerify is verified against exactly that key over the full handshake
transcript, ClientHello through ClientKeyExchange.

Two boots of the Docker lab produced different CA key fingerprints, consistent
with Fortuna seeded from `/dev/urandom`. This rules out exact deterministic
repetition in the reproduced environment; it says nothing about entropy quality
in the historical BHyve/Solo5 deployment.

The Marvin-class timing side-channel on port 443 is theoretically concerning.
The RSA decryption uses Zarith `powm` with value-dependent timing. RSA
blinding is intended to decorrelate the exponentiation from the attacker's
ciphertext, but Marvin-class leakage can still occur during deblinding,
integer-to-byte conversion, or padding handling. A measurement campaign would
be research-grade and I did not attempt one. The secret-bearing ports require
client authentication before the RSA decryption step, so a direct oracle is
not reachable there, but a successful private-key-operation attack against the
:443 key — whose certificate is signed by the same CA — could potentially
compose with the missing server-side EKU check on port 10000. I did not
investigate this composition.

## Reproducing the findings

The lab repository is at
[github.com/matthiasgoergens/btc-pinata-lab](https://github.com/matthiasgoergens/btc-pinata-lab).

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

The connection dies. The unikernel stays up. Five bytes, no authentication,
nearly twelve years.

## Audit result

The `mirleft/ocaml-tls` authors set out to build a TLS stack that was "not
quite so broken." The Piñata was their public bet that they had succeeded.
After checking both models' hypotheses against the source and reproducing each
concrete finding, I found no chain-validation or CertificateVerify bypass in
the pinned TLS and X.509 code.

The audit did find a crash bug present since the initial release, an
exponential DoS still in current upstream, and a missing-purpose-check that
shipped as a CVE, but none bypassed the mutual-TLS authentication protecting
the key.

For a roughly 5,000-line TLS library written in a memory-safe language by a
small team, that is a remarkable result. This was an empirical adversarial
audit rather than a formal verification; against the concrete goal of bypassing
client authentication, the stack held up.

The owners withdrew the 10 BTC in 2018. This 2026 audit provides stronger
evidence that the pinned stack resisted the authentication bypass the challenge
invited.
