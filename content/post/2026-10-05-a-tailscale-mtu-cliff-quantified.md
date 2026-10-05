---
title: A Tailscale MTU Cliff, Quantified
description: "SSH hangs while ping flies? Small packets pass and large packets die — a classic black hole. How to measure the exact cliff with a DF ping sweep, and how to make the fix stick."
date: 2026-10-05T12:10:00+08:00
tags: [Networking, Tailscale, WireGuard, Troubleshooting]
---

## The Symptom: "Online" Services That Hang

Two sites, connected via Tailscale. Small requests fly: the identity endpoint responds in 20 ms, single pings pass. But anything with a *large* response — an SSH handshake, a JSON list of media sections — hangs forever. The connection looks alive. It isn't.

Classic black-hole network path: **small packets pass, large packets die**. The tricky part is that "small packets pass" is exactly what makes it look like a configuration problem on your end, not the network.

## Measure the Cliff, Don't Guess

Tailscale's interface MTU is 1280 by default, which means the inner packet plus WireGuard overhead (roughly +60 bytes) must survive the UDP path. To map the cliff, send DF-set pings of increasing inner size and watch where loss begins:

| Inner payload | WireGuard-wrapped | Loss |
|---|---|---|
| 1200 B | ~1260 B | 0% |
| 1230 B | ~1290 B | 66% |
| 1272 B | ~1332 B | 100% |

The cliff sits between **1260 and 1290 wrapped bytes** — some middlebox on the path drops encapsulated packets above that threshold. Everything TCP tries to push a full-size segment into that gap and stalls.

Two measurement gotchas worth their weight in gold:

1. **Packets larger than MTU-28 get rejected locally** ("message too long") — that's your kernel refusing, not evidence about the path.
2. Always set `-M do` (DF). Fragmentation will happily lie to you about path behavior.

## The Fix: Drop the Interface to 1200

```bash
ip link set dev tailscale0 mtu 1200
```

TCP MSS automatically converges to 1160. After this: SSH transfers 1 MB in 1.45 s, API calls 0.02 s, streaming stable at the real uplink rate.

But `tailscaled` **resets the MTU back to 1280 on reconnect** — so a one-shot command rots. Persist it:

- A systemd timer that idempotently re-sets MTU every 2 minutes;
- A watchdog (every 3 min, rate-limited to once per 30 min) that detects the black-hole signature — small pings pass, ≥50% loss on 1200 B DF probes — and bounces the tunnel to renegotiate endpoints.

## The Takeaway

"Everything works except the big things" is an MTU problem until proven otherwise. Quantify the cliff with a DF ping sweep before touching any application config — five minutes of scanning saves hours of blaming the wrong layer.
