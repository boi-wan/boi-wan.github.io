---
date: "2026-07-09"
draft: true
title: "Thread on a Budget"
description: "How I got a standalone OpenThread Border Router talking to a network-attached Thread radio, without HAOS, without a Supervisor, and local-first."
summary: "Wiring a SONOFF Dongle Max into a Dockerized Home Assistant"
tags: ["Thread", "Home Assistant", "home lab"]
---

This is my very first post on this blog, and it's a bit of a doozy. It's about Thread, the low-power mesh networking protocol that underpins Matter, and how I got it working in my home lab without the official Home Assistant OS add-on.

## Context

My 8th-generation Intel NUC handles networking and light services in my home lab, through Docker Compose. I recently started playing with Home Assistant (HA), which I deliberately chose to run as a container, not Home Assistant OS (HAOS). Since I run everything else through Docker Compose and didn't want a second orchestration model just for HA.

That choice has one real cost: no Supervisor means no add-on ecosystem. Which matters a lot the moment you want Thread and Matter support, because the "official" path of installing the OpenThread Border Router (OTBR) OS add-on, simply isn't available to you. Everything from there has to be assembled by hand.

## The hardware: SONOFF Dongle Max

I picked the SONOFF Dongle Max (EFR32MG24 + ESP32, PoE/Ethernet capable) during Amazon Prime Day over the more commonly recommended Home Assistant Connect ZBT-2, mainly on price and on the PoE option. Reviews are very positive, and the device is well-supported in the OpenThread ecosystem. It also has a USB-C connector, which is a nice touch.
The idea of a radio that doesn't need to live wherever my NUC happens to sit was appealing for a mesh network, where placement matters for coverage.

That "network-attached radio" idea turned out to be the crux of everything that follows.

## Architecture: OTBR has to stand alone

With no Supervisor, OTBR needs to run as its own Docker service on my NUC, independent of the HA container, communicating with it purely over its REST API. Conceptually simple. In practice, this is where things got interesting.

### First Attempt: raw `openthread/border-router` + a hand-rolled socat bridge

OTBR doesn't natively speak TCP to a radio because it expects a serial device. Since the Dongle Max would be reachable over the network (static IP, TCP port 6638), the plan was: run `socat` in one container to bridge the TCP connection to a pseudo-terminal, and point OTBR's `OT_RCP_DEVICE` at that pty in a second container, sharing state through a bind-mounted `/tmp`.

Guess what? It didn't work. The pty socat created was visible by path, but not as a working character device, from OTBR's container. Each container normally gets its own private devpts instance, even when `/tmp` is shared. A pty allocated in one container's mount namespace simply isn't a usable device node in another's, regardless of whether the symlink resolves.

There's a real fix for this (bind-mounting the host's actual `/dev/pts` into both containers instead of relying on `--device`, plus `--privileged` on both), but before going further down that road, a better option surfaced.
