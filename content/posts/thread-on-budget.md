---
date: "2026-09-15"
draft: false
title: "Thread on a Budget"
description: "How I got a standalone OpenThread Border Router talking to a network-attached Thread radio. Local-first."
summary: "Wiring a SONOFF Dongle Max into a Dockerized Home Assistant"
tags: ["Thread", "Home Assistant", "home lab"]
---

This is the first post on this blog. It's about Thread, the low-power mesh protocol behind Matter, and how I got it working in my home lab without Home Assistant OS.

## Context

I recently started thinking about getting rid of third party services in my smart home (like IKEA's DIRIGERA cloud), aiming to find a way to run everything locally. I want to avoid vendor lock-in, and I want to be able to control my devices even if the cloud goes down or the company disappears. So, I started looking at Home Assistant and Matter.

There are two ways to run Home Assistant: the full Home Assistant OS (HAOS) with a Supervisor, or the Home Assistant container image (HA) on top of an existing OS.

Since my homelab's 8th-gen Intel NUC is running networking and a few other base services with Docker on top of Debian 13, the choice was obvious: run HA in a container as well. There is a but: HAOS is the only supported path for add-ons, and Thread/Matter support is only available as an add-on.

## The hardware: SONOFF Dongle Max

Reddit people usually recommend the _Home Assistant Connect ZBT-2_, which sits around 65 EUR. During Amazon Prime Day I found a good deal for the _SONOFF Dongle Max_ (EFR32MG24 + ESP32, PoE/Ethernet capable): 35 EUR was a good price for a Thread radio that can also work as a Matter controller. Reviews are positive and the chip is well supported in the OpenThread ecosystem. In addition, the Dongle Max is network-attached, so it can sit anywhere on the LAN and doesn't have to be physically next to any server.

That same network-attached design is also the source of most of the headache below.

## Architecture: OTBR has to stand alone

With no HA Supervisor, OTBR runs as its own Docker service on the server, separate from the HA container, and the two talk over OTBR's REST API.

### First attempt: raw `openthread/border-router` + a socat bridge

TLDR: it didn't work.

OTBR does expect a serial device; it doesn't speak TCP to a radio. Since the Dongle Max is reachable over the network (with a static IP, usually TCP port 6638), the plan was:

- run `socat` in one container to bridge the TCP connection to a pseudo-terminal
- point OTBR's `OT_RCP_DEVICE` at that `pty` from a second container, sharing state through a bind-mounted `/tmp`

It didn't work. The `pty` created by socat was visible by path from the OTBR container, but not usable as a character device. Each container gets its own private `devpts` instance, even when `/tmp` is shared, so a `pty` allocated in one container's mount namespace is not a valid device node in another, no matter if the symlink resolves.

There is a fix (bind-mount the host's `/dev/pts` into both containers and run both privileged), but before going down that road, a better option showed up.

### Second attempt: `bnutzer/otbr-tcp`

TLDR: it worked, with some small config tweaks.

This is a community image built exactly for this scenario: OTBR next to a Dockerized Home Assistant. The key difference is that socat and `otbr-agent` run **inside the same container**, so the devpts namespace problem does not exist. It also supports both a network-attached radio (`RCP_HOST`/`RCP_PORT`) and a local USB stick, so switching transport later is a one-variable change.

This is what I ended up running.

## Debugging

Getting from "container starts" to "it actually works" took three fixes. All of them small config details, none obvious from the logs.

### Bug 1: TREL couldn't bind to `eth0`

First boot ended in an infinite restart loop:

```shell
[C] P-Trel--------: Failed to bind socket to the interface eth0
[C] Platform------: PrepareSocket() at trel.cpp:215: No such device
socat[44] N read(5, ...): Input/output error (probably PTY closed)
```

The RCP connection over TCP was actually fine. The fatal error came from a second, unrelated link. OTBR also opens a TREL (Thread Radio Encapsulation over IPv6) socket on the host's backbone interface, and the image defaults it to `eth0`. Debian 13 uses predictable interface naming, so that name doesn't exist on this server; the real interface is `eno1`. To confirm the right interface name, run:

```shell
ip -4 addr show | grep <host-ip>
```

and override the default interface in the compose file with

```yaml
environment:
  OTBR_BACKBONE_IF: "eno1"
```

### Bug 2: Port collisions with existing services

Once TREL bound correctly, the agent came up and immediately hit:

```shell
[CRIT]-WEB-----: Failed to start web server on 0.0.0.0:8080: Address already in use (errno 98)
```

The server already has a full port map (Glance on 8080, Pi-hole on 8081, WUD on 3000, Dockhand on 3001, and so on). OTBR's web UI defaults to 8080 and its REST API to 8081, both already taken. I moved both, plus one extra variable so the bundled web UI knows where the REST API went:

```yaml
environment:
  OTBR_REST_LISTEN_PORT: "8082"
  OTBR_WEB_PORT: "8090"
  OTBR_WEB_PATCH_REST_PORT: "1"
```

### Bug 3: A one-letter typo in an environment variable

The REST API has no authentication, so I wanted it bound to loopback only and set `OTBR_REST_LISTEN_ADDRESS: 127.0.0.1`. It silently did nothing. The image's Dockerfile has the answer:

```shell
ENV OTBR_REST_LISTEN_ADRESS="0.0.0.0"
```

The variable name is misspelled in the image itself, missing a "D". I left a comment in the compose file, so a future edit doesn't quietly re-expose an unauthenticated REST API to the whole LAN.

### One warning I ignore

Every boot also logs:

```shell
ipset v6.34: Kernel support protocol versions 6-7 while userspace supports protocol versions 6-6
The set with the given name does not exist
```

This is OTBR initializing the firewall rules for border routing: the set "does not exist" because OTBR is about to create it. Benign, safe to ignore.

## The working configuration

```yaml
services:
  otbr:
    image: bnutzer/otbr-tcp:latest
    container_name: otbr
    network_mode: host
    restart: unless-stopped
    privileged: true
    devices:
      - /dev/net/tun
    environment:
      RCP_HOST: "192.168.0.104" # SONOFF Dongle Max, static IP
      RCP_PORT: "6638"
      RCP_BAUDRATE: "115200" # Dongle Max specific; not the image's 460800 default
      OTBR_THREAD_IF: "wpan0"
      OTBR_BACKBONE_IF: "eno1" # real interface name — check yours, don't assume eth0
      OTBR_REST_LISTEN_ADRESS: "127.0.0.1" # note: typo is in the image itself
      OTBR_REST_LISTEN_PORT: "8082"
      OTBR_WEB_ENABLE: "1"
      OTBR_WEB_PORT: "8090"
      OTBR_WEB_PATCH_REST_PORT: "1"
    volumes:
      - /app/data/stacks/durin-doors/otbr:/var/lib/thread
    labels:
      wud.watch: "false" # image ships weekly latest-only builds, no semver
```

A working boot ends like this:

```shell
[INFO]-REST----: RestWebServer listening on 127.0.0.1:8082
[INFO]-APP-----: Radio Co-processor version: SL-OPENTHREAD/2.4.5.0_GitHub-797150858; EFR32; ...
otbr-agent is ready.
[INFO]-WEB-----: Border router web started on wpan0
```

## Wiring it into Home Assistant

With OTBR healthy, the HA side is straightforward:

1. **Settings → Devices & Services → Add Integration → Open Thread Border Router**, pointing at `http://127.0.0.1:8082`. HA and OTBR both run with `network_mode: host` on the same NUC, so loopback works between them.
2. **Add the Thread integration** and explicitly create a new network, instead of letting anything auto-join. OTBR's TREL discovery had already picked up a neighboring Thread network (almost certainly a nearby Nest/Google Home border router), so check the Thread panel afterwards and confirm the network shown is actually yours.
3. **Confirm the Matter integration** is present.

## Validate persistence before pairing devices

Before pairing a single bulb, I restarted the `otbr` container and re-checked the Thread panel in HA. The point is to verify that `/var/lib/thread` really persists the Thread dataset across container recreates, and that the network identity is not quietly regenerated each time. Finding a broken volume mount now takes five minutes; finding it after six devices are paired to a network that no longer exists takes much more.

It passed: same network, same credentials, after a full container restart.

## What's next

With a persistent border router in place, commissioning Matter devices is easy and local-first.

## Links

- [Open Thread Border Router](https://openthread.io/guides/border-router)
- [Home Assistant Thread Integration](https://www.home-assistant.io/integrations/thread/)
- [SONOFF Dongle Max](https://amzn.eu/d/0bJT5eAH)
- [docker-otbr-tcp](https://github.com/bnutzer/docker-otbr-tcp)
