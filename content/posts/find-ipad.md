+++
title = "Find My said 'Nearby'. The iPad said nothing."
date = 2026-07-30T21:30:00+08:00
description = "My iPad mini went missing in my own house. Find My could see it but couldn't make it beep. One prompt to Claude produced a Bluetooth Geiger counter that found it — and the postmortem explains why Play Sound never stood a chance."
+++

This one was, essentially, a one-shot AI solution, so let me lead with that.
I told Claude Code: Find My says my iPad mini is in the house, I've pressed
Play Sound, I can't hear anything — can we do Bluetooth, make the MacBook
beep hotter and colder as I move towards or away from it? It wrote 266 lines
of Swift around CoreBluetooth. The first time macOS let the binary touch the
radio, the first device row it printed was my iPad, by name. The code is at
[github.com/matthiasgoergens/find-ipad](https://github.com/matthiasgoergens/find-ipad)
— one file, no dependencies, `swiftc -O find-ipad-scan.swift -o find-ipad-scan`
and you have it too.

To keep "one-shot" honest: there were exactly two iterations, neither of them
debugging. Mid-flight I asked for a scrolling textual log alongside the
beeping, because I didn't trust my ears to integrate a Geiger counter — that
went in before the first successful scan. And the first run found zero
devices, which turned out to be macOS withholding Bluetooth permission from
the terminal, fixed in System Settings rather than in code. The program
itself never produced a wrong result that needed fixing. I have spent
multi-day debugging sessions on smaller programs than this.

## The taunt

The situation, before the tool: Find My showed the iPad at home, tagged
**Nearby**, location updated two minutes ago and refreshing faithfully every
few minutes. Battery evidently fine. Play Sound: nothing, over hours, over
many attempts. I had been everywhere in the house with my ear out. The
sidebar might as well have read "warm, getting warmer" while declining to say
anything further.

"Nearby", it turns out, is the load-bearing word. It means *this Mac can hear
the iPad over Bluetooth right now*. Apple devices broadcast BLE
advertisements constantly — it's how the Find My network locates devices that
are offline, with every passing iPhone acting as a relay. Which means the
signal is right there in the air, with a strength that falls off with
distance, and any laptop with a Bluetooth radio can play hot-and-cold with
it. There's no UWB precision-finding between a Mac and an iPad mini (neither
has the chip), no hidden API — but received signal strength is free.

## The tool

The scanner has two modes. List mode shows every Apple device in radio
range, sorted by smoothed RSSI, with Apple's manufacturer-data frame types
decoded — and here the identification problem, which I had expected to be
the hard part, simply evaporated. BLE advertisements rotate their addresses
every fifteen minutes for anti-tracking reasons, so I'd braced for
guess-and-check against anonymous IDs. Instead, because the iPad is on my
own Apple ID, macOS resolved its identity outright:

```
 RSSI  SIGNAL                    ID        TYPES              NAME
  -67  ███████████░░░░░░░░░░░░░  8154CC68  NearbyInfo         Matthias's iPad mini
  -69  ██████████░░░░░░░░░░░░░░  8C9E0F27  FindMy             —  ◀ FindMy beacon
```

There it is. Row one, by name, at −67 dBm from the laptop's resting spot.
(The anonymous `FindMy` beacons below it were something else — an earbuds
case, or a neighbour's AirTag bleeding through the wall. Didn't matter;
didn't need them.)

Track mode locks onto one ID and beeps like a Geiger counter — faster and
louder as the signal strengthens — while printing a timestamped log line
every second with the exact reading and a warmer/colder trend, computed
against a few seconds ago:

```
18:04:12  [8154CC68]   -77 dBm  ███████░░░░░░░░░░░░░░░░░  ▲ WARMER (+6 dB)   Matthias's iPad mini
18:04:27  [8154CC68]   -80 dBm  ██████░░░░░░░░░░░░░░░░░░  ▼ colder (-9 dB)   Matthias's iPad mini
```

The log immediately earned its keep: my first walk peaked at −71 dBm and I
sailed straight through the warm zone and out the far side, down to −83,
beeps slowing behind me while I confidently marched on. The trail in the
terminal didn't argue; it just sat there reading −71 → −75 → −77 → −80 → −83.
Turned around, went back to the peak, searched outward from there.

One physics note, because it surprised me on the floor and the explanation is
tidy: I'd expected an inverse-square law to make the last two metres
dramatic, and they weren't. dBm is a log scale, so inverse-square flattens to
a constant −6 dB per doubling of distance — 10 cm to 2 m is only ~26 dB in
theory, and in practice receiver saturation, near-field weirdness inside one
wavelength (12.5 cm at 2.4 GHz), and your own body swinging readings by
10–20 dB compress it further. RSSI tells you which room, not which cushion.
That's exactly why AirTag precision finding uses UWB time-of-flight instead.
For the last metre, you use your eyes.

## The anticlimax

Which brings me to where it was: on the floor. In the open. A black iPad in
a black case, flat on a dark brown wooden floor, in poor evening light —
and I'm colourblind. I had walked past it, probably repeatedly, possibly
within arm's reach, while a laptop beeped at me that I was practically
standing on it. The Bluetooth was right. The failure was optical. The
cheapest upgrade to Find My, I now understand, is a case in a colour that
doesn't occur in flooring.

## Why Play Sound never had a chance

The postmortem turned out to be the most interesting part, because every
obvious explanation died on evidence.

*It was on silent?* No — Find My's Play Sound deliberately bypasses mute and
volume, precisely because a lost device is disproportionately likely to be a
muted one.

*It was offline, and the sound would play when it reconnected?* Partly —
but then it should have chirped when I picked it up and woke it. It didn't.
And my own screenshot from mid-hunt shows Find My reporting "Play Sound:
Off" — not pending — after multiple attempts.

*No connectivity path?* This is where it gets genuinely annoying. This is a
cellular iPad. The SIM works; the data plan is alive and in regular use. It
had 44% battery when found. Top floor, decent coverage. Every hop of the
delivery path was individually healthy.

The picture that survives all the evidence: **findable and reachable are
different channels, and a sleeping iPad keeps only the first.** Location
comes from the iPad's BLE beacon, relayed by my other Apple devices —
transmit-only, works in deep sleep, hence the cheerful "2 minutes ago". But
Play Sound is a push command that needs an *inbound* data path, and an iPad
in extended deep sleep drops its WiFi association and, evidently, lets the
cellular data session lapse too. An iPhone can't do that — it's contractually
a telephone, it must be reachable for calls. An iPad has no such obligation
and economises accordingly. My daughter's daily use never sees this, because
she uses it awake; the gap only opens when nobody is touching it, which is
precisely and exclusively when Find My needs it.

And the queued commands? Apparently written in disappearing ink. Whether
they expire server-side or never durably queue, by the time the iPad finally
reconnected there was nothing waiting for it. So for a deeply sleeping iPad,
Play Sound isn't delayed — it's *lossy* — and nothing in the UI hints at
that. The button clicks. The map says Nearby. The sound never comes.

## The design that already exists

The obvious question — my Mac is within Bluetooth earshot of the iPad, both
signed into my Apple ID, why can't the beep command travel over *that* radio?
— turned into a satisfying back-and-forth with Claude in which I proposed
fixes and it explained what each one breaks, and I fixed the breakage.

Replay attacks against a naive signed beep packet? Sign a timestamp — if the
iPad has battery enough to beep, it has a clock. Listening costs power and
the beacon is transmit-only by design? Duty-cycle it; I'll happily let my
laptop blast for ten minutes until the iPad's next listening slot — and BLE
already opens a listen window right after each connectable advertisement, so
the rendezvous is nearly free. Want challenge-response freshness without a
clock? Put a nonce in the beacon the iPad already broadcasts every two
seconds, and sign the command over that — taking care the nonce rotates in
lockstep with the address, or you've just built the tracking identifier the
rotation exists to prevent.

At which point we both observed that I had blindly groped my way to roughly
the Find My accessory protocol — the thing an AirTag already implements,
beaconing and listening and verifying owner commands for a year on a coin
cell holding about a thirtieth of an iPad mini's battery. Apple has the
keys, the silicon, the shipped precedent. iPads still don't do it, and
that's a product decision — presumably some mix of controller-firmware
gating on newer chips, support-matrix sprawl, and a spreadsheet that says
the scenario is rare. Having spent an evening as the scenario, I'd like a
word with the spreadsheet: the official mechanism delivered nothing, not
late but never, while the unofficial mechanism was a prompt and a `swiftc`
invocation away.

So: if a device with a Bluetooth radio ever goes missing in your house while
Find My smugly reports it Nearby —
[the scanner is right here](https://github.com/matthiasgoergens/find-ipad).
Documented track record: one (1) iPad found. Success rate so far: 100%.
